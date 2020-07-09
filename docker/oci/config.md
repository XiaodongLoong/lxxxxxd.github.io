翻译自：https://github.com/opencontainers/runtime-spec/blob/master/config.md

这个配置文件包含了在容器上实现标准操作所必要的元数据。这包括了执行集成，环境变量注入，沙箱特性的使用。

公认的的模板在这篇文档中将被定义，但是这里有一个json格式的模板[`schema/config-schema.json`](https://github.com/opencontainers/runtime-spec/blob/master/schema/config-schema.json)和go的二进制模板[`specs-go/config.go`](https://github.com/opencontainers/runtime-spec/blob/master/specs-go/config.go)。平台相关的配置模板在[`platform-specific documents`](https://github.com/opencontainers/runtime-spec/blob/master/config.md#platform-specific-configuration)进行了定义。对于某些属性只在一些平台上进行了定义，Go的属性有一个 platform 标签列举了这些协议。
下面是配置格式每一个属性的详细的描述和有效值。平台相关的配置值，定义在下面的 平台相关配置部分。

### 声明版本
**ociVersion** (string,REQUIRED) MUST 必须以 [`SemVer 2.0.0`](https://semver.org/spec/v2.0.0.html)格式声明OCI运行时bundle所符合的版本。开放容器倡议运行时规范遵循语义版本控制，并在主要版本中保留了向前和向后的兼容性。 例如，如果配置符合该规范的1.1版，则它与支持此规范的任何1.1或更高版本的所有运行时兼容，但与支持1.0而不是1.1的运行时不兼容。

`"ociVersion": "0.1.0"`

## <a name="configRoot" />Root

**root** (object, OPTIONAL) 声明了容器的根文件系统。在Windows上， Windows Server容器的这个属性是REQUIRED,对于[`Hyper-V 容器`](https://github.com/opencontainers/runtime-spec/blob/master/config-windows.md#hyperv)是 MUST NOT。

其他的平台，这个属性是REQUIRED
* **`path`** (string, REQUIRED) 定义容器的根文件系统的路径.
    * Windows 平台, `path` MUST 必须是 [volume GUID path][naming-a-volume].
    * POSIX 平台, `path` 要么是绝对路径，要么是相对路径.
        例如, 有一个bundle路径为 `/to/bundle`， 一个根文件系统为 `/to/bundle/rootfs`, `path` 值可以是 `/to/bundle/rootfs`，也可以是 `rootfs`.
        通常应该 SHOULD 是 `rootfs`.

    path声明的路径必须存在 MUST.

* **`readonly`** (bool, OPTIONAL) true 代表根文件系统在容器里面必须 MUST 是 read-only, 默认值是false.
    * Windows 平台, 这个值必须 MUST 忽略或者false.

### Example (POSIX platforms)

```json
"root": {
    "path": "rootfs",
    "readonly": true
}
```

### Example (Windows)

```json
"root": {
    "path": "\\\\?\\Volume{ec84d99e-3f02-11e7-ac6c-00155d7682cf}\\"
}
```

## <a name="configMounts" />Mounts

**`mounts`** (array of objects, OPTIONAL) 除了 [`root`](#root)外，定义了附加的挂载.
运行时 MUST 按照列表顺序执行挂载.
Linux平台, 参数见文档 [mount(2)][mount.2] 系统调用 man 手册.
Solaris平台, 挂载项 对应 'fs' 资源，参考 [zonecfg(1M)][zonecfg.1m] man 手册.

* **`destination`** (string, REQUIRED) 目标挂载点:容器内的路径.
    这个值 MUST 是 绝对路径.
    * Windows: 一个目标挂载 MUST NOT 必须不嵌套在其他挂载中 (e.g., c:\\foo and c:\\foo\\bar).
    * Solaris: "dir"对应的 fs 资源在 [zonecfg(1M)][zonecfg.1m].
* **`source`** (string, OPTIONAL) 设备名称, 但是也可以是一个用于绑定挂载（bind mount）的或者虚拟的文件或者目录名称.
    绑定挂载的路径，到bundle的绝对路径或相对路径
    绑定挂载（bind mount—）-- 拥有 `bind` 或 `rbind` 选项参数.
    * Windows平台: 容器本地主机上的一个路径. 不支持 UNC路径和驱动映射（mapped drives）.
    * Solaris平台: 在 [zonecfg(1M)][zonecfg.1m] 中fs资源对应的“special”.
* **`options`** (array of strings, OPTIONAL) 要用的文件系统的挂载选项.
    * Linux: [mount(8)][mount.8] man 手册列出.
      注意 [filesystem-independent][mount.8-filesystem-independent] 和 [filesystem-specific][mount.8-filesystem-specific] 选项都被列出来了.
    * Solaris: 在 [zonecfg(1M)][zonecfg.1m]中的fs资源选项中对应的“options”.
    * Windows: 运行时必须 MUST 支持 `ro`, 当给定`ro`时以只读模式挂载文件系统.

### Example (Windows)

```json
"mounts": [
    {
        "destination": "C:\\folder-inside-container",
        "source": "C:\\folder-on-host",
        "options": ["ro"]
    }
]
```

### <a name="configPOSIXMounts" />POSIX平台 Mounts

对于POSIX平台， `mounts` 结构具有如下属性：

* **`type`** (string, OPTIONAL) 被挂载的文件系统类型.
    * Linux: */proc/filesystems* 列出的内核支持的文件系统类型 (e.g., "minix", "ext2", "ext3", "jfs", "xfs", "reiserfs", "msdos", "proc", "nfs", "iso9660"). 绑定挂载 (`options` 包含 `bind` 或 `rbind`), 类型是虚拟的, 通常是 "none" (不在 */proc/filesystems*).
    * Solaris: fs 资源对应的"type"，参考 [zonecfg(1M)][zonecfg.1m].

### Example (Linux)

```json
"mounts": [
    {
        "destination": "/tmp",
        "type": "tmpfs",
        "source": "tmpfs",
        "options": ["nosuid","strictatime","mode=755","size=65536k"]
    },
    {
        "destination": "/data",
        "type": "none",
        "source": "/volumes/testing",
        "options": ["rbind","rw"]
    }
]
```

### Example (Solaris)

```json
"mounts": [
    {
        "destination": "/opt/local",
        "type": "lofs",
        "source": "/usr/local",
        "options": ["ro","nodevices"]
    },
    {
        "destination": "/opt/sfw",
        "type": "lofs",
        "source": "/opt/sfw"
    }
]
```

## <a name="configProcess" />Process

**`process`** (object, OPTIONAL) 指定具体的容器进程.
当 [`start`](runtime.md#start) 方法调用的时候，这个属性是 REQUIRED

* **`terminal`** (bool, OPTIONAL) 指定一个终端是否attach到这个进程，默认是false
    比如说,在Linux系统上如果设置为true，一个虚拟终端对被分配出来，slave虚拟终端是这个进程的标准输入 [standard streams][stdin.3]的副本.
* **`consoleSize`** (object, OPTIONAL) 指定控制台字符大小.
    如果`terminal` 是 `false`或没有设置，运行时必须忽略`consoleSize` .
    * **`height`** (uint, REQUIRED)
    * **`width`** (uint, REQUIRED)
* **`cwd`** (string, REQUIRED) 可执行文件的工作路径.
    这个值必须 MUST 是一个绝对路径.
* **`env`** (array of strings, OPTIONAL) 环境变量，和 [IEEE Std 1003.1-2008's `environ`][ieee-1003.1-2008-xbd-c8.1]具有相似的语义.
* **`args`** (array of strings, OPTIONAL) 参数，和 [IEEE Std 1003.1-2008 `execvp`'s *argv*][ieee-1003.1-2008-functions-exec]具有相似的语义.
    这个规范都继承于 IEEE 标准，至少需要（ REQUIRED ）一个执行入口(non-Windows), 并且这个执行入口和 `execvp`的 *file*具有相似的语义规范. 在Windows上这是可选的,如果这个属性被忽略， `commandLine` 是 REQUIRED.
* **`commandLine`** (string, OPTIONAL) 在windows上指定了要执行的全命令行.
    在 Windows上，推荐使用这种方式来提供命令行. 如果被忽略，在将系统调用到Windows之前，运行时将退回到转义和连接来自`args`的字段.

### <a name="configPOSIXProcess" />POSIX process

支持 POSIX rlimits的系统， (比如 Linux 和 Solaris),  `process` 对象支持下述的详细进程属性:

* **`rlimits`** (array of objects, OPTIONAL) 允许设置进程的资源限制.
    每一个项具有如下的结构：

    * **`type`** (string, REQUIRED) 被限制的平台资源.
        * Linux: valid values are defined in the [`getrlimit(2)`][getrlimit.2] man page, such as `RLIMIT_MSGQUEUE`.
        * Solaris: valid values are defined in the [`getrlimit(3)`][getrlimit.3] man page, such as `RLIMIT_CORE`.

        对于不能映射到相关的内核接口的值，运行时 MUST产生一个错误 [generate an error](runtime.md#errors).
        对于 `rlimits`的每一项,  [`getrlimit(3)`][getrlimit.3] 系统调用在 `type` MUST 成功.
        对于下述的属性, `rlim` 指的是强制的 `getrlimit(3)` 系统调用的返回.

    * **`soft`** (uint64, REQUIRED) 强制限制对应资源的值.
        `rlim.rlim_cur` MUST 必须与配置的值匹配.
    * **`hard`** (uint64, REQUIRED) soft 限制所能到的最大值，可以被一个非特权进程设置.
        `rlim.rlim_max` MUST 必须与配置的值相匹配.
        只有特权进程 (比如： 配置 `CAP_SYS_RESOURCE` 能力集合) 可以具有 hard limit.

    如果`rlimits` 包含同一个`type`项的冗余副本, 运行时必须 MUST产生一个错误 [generate an error](runtime.md#errors).

### <a name="configLinuxProcess" />Linux Process

对于基于 Linux的系统,  `process` 对象支持下述详细进程属性.

* **`apparmorProfile`** (string, OPTIONAL) 为进程指定 AppArmor profile 文件.
    更多AppArmor详细信息, 查看 [AppArmor documentation][apparmor].
* **`capabilities`** (object, OPTIONAL) 指定进程能力集合的对象数组.
     [capabilities(7)][capabilities.7] man 手册中定义的能力值, 比如 `CAP_CHOWN`.
    不能映射到相应的内核接口的值必须 MUST 产生一个错误.
    `capabilities` 包含下述属性:

    * **`effective`** (array of strings, OPTIONAL)  进程持有的有效的能力集.
    * **`bounding`** (array of strings, OPTIONAL)  进程持有的边界能力集合.
    * **`inheritable`** (array of strings, OPTIONAL) 进程持有的可继承的能力集.
    * **`permitted`** (array of strings, OPTIONAL) 进程持有的允许的能力集合
    * **`ambient`** (array of strings, OPTIONAL) 进程持有的周边能力集合.
* **`noNewPrivileges`** (bool, OPTIONAL) 防止进程获取另外的权限
    比如说,  [`no_new_privs`][no-new-privs] linux内核文档有`prctl`系统调用如何实现的信息.
* **`oomScoreAdj`** *(int, OPTIONAL)* 调整 oom-killer 分数， `[pid]/oom_score_adj` `[pid]`进程的编号， [proc pseudo-filesystem][proc_2].
    一旦`oomScoreAdj`设置, 运行时必须 MUST 设置 `oom_score_adj`为给定的值.
    一旦`oomScoreAdj` 设置, 运行时不必 MUST NOT 改变 `oom_score_adj`的值.

    这是对每一个进程都进行设置的, 如 [`disableOOMKiller`](config-linux.md#memory) 所示，是memory cgroup的范围.
    更多关于这两个设置一起工作的信息，见 [the memory cgroup documentation section 10. OOM Contol][cgroup-v1-memory_2].
* **`selinuxLabel`** (string, OPTIONAL) 为进程指定 SELinux 标签.
    更多关于SELinux信息, 见 [SELinux documentation][selinux].
    
    ### <a name="configUser" />用户

进程的用户是一个平台相关的结构，允许特定控制进程以什么用户运行.

#### <a name="configPOSIXUser" />POSIX-platform User

对于POSIX 平台， `user`结构具有如下的属性:

* **`uid`** (int, REQUIRED) 指定用户的唯一标识，见 [container namespace](glossary.md#container-namespace).
* **`gid`** (int, REQUIRED) 指定用户组唯一标识，见 [container namespace](glossary.md#container-namespace).
* **`umask`** (int, OPTIONAL) 指定用户的 [umask][umask_2] .如果不指定，umask不应该从调用进程被改变 .
* **`additionalGids`** (array of ints, OPTIONAL) 指定附加的用户组ID（会添加到进程），见 [container namespace](glossary.md#container-namespace).

_Note: uid 和gid 的符号名称, 比如uname 和 gname,分别留给上层推导 (比如. `/etc/passwd` 解析, NSS, 等)_

### Example (Linux)

```json
"process": {
    "terminal": true,
    "consoleSize": {
        "height": 25,
        "width": 80
    },
    "user": {
        "uid": 1,
        "gid": 1,
        "umask": 63,
        "additionalGids": [5, 6]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "sh"
    ],
    "apparmorProfile": "acme_secure_profile",
    "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
    "noNewPrivileges": true,
    "capabilities": {
        "bounding": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
       "permitted": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
       "inheritable": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL",
            "CAP_NET_BIND_SERVICE"
        ],
        "effective": [
            "CAP_AUDIT_WRITE",
            "CAP_KILL"
        ],
        "ambient": [
            "CAP_NET_BIND_SERVICE"
        ]
    },
    "rlimits": [
        {
            "type": "RLIMIT_NOFILE",
            "hard": 1024,
            "soft": 1024
        }
    ]
}
```
### Example (Solaris)

```json
"process": {
    "terminal": true,
    "consoleSize": {
        "height": 25,
        "width": 80
    },
    "user": {
        "uid": 1,
        "gid": 1,
        "umask": 7,
        "additionalGids": [2, 8]
    },
    "env": [
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "TERM=xterm"
    ],
    "cwd": "/root",
    "args": [
        "/usr/bin/bash"
    ]
}
```

#### <a name="configWindowsUser" />Windows 用户

对于基于Windows的系统，user结构具有如下的属性：

* **`username`** (string, OPTIONAL) 为进程指定了用户名.

### Example (Windows)

```json
"process": {
    "terminal": true,
    "user": {
        "username": "containeradministrator"
    },
    "env": [
        "VARIABLE=1"
    ],
    "cwd": "c:\\foo",
    "args": [
        "someapp.exe",
    ]
}
```


## <a name="configHostname" />Hostname 主机名称

* **`hostname`** (string, OPTIONAL) 指定容器内进程运行可见的容器主机名称.
    在Linux上, 举个例子， 这个会改变容器内的主机名称，uts 命名空间 [container](glossary.md#container-namespace) [UTS namespace][uts-namespace.7].
    依赖命名空间配置 [namespace configuration](config-linux.md#namespaces), 容器 UTS 命名空间可能是 [runtime](glossary.md#runtime-namespace) [UTS namespace][uts-namespace.7].

### Example

```json
"hostname": "mrsdalloway"
```

## <a name="configPlatformSpecificConfiguration" />Platform-specific configuration 平台相关配置

* **`linux`** (object, OPTIONAL) [Linux-specific configuration](config-linux.md).
    This MAY be set if the target platform of this spec is `linux`.
* **`windows`** (object, OPTIONAL) [Windows-specific configuration](config-windows.md).
    This MUST be set if the target platform of this spec is `windows`.
* **`solaris`** (object, OPTIONAL) [Solaris-specific configuration](config-solaris.md).
    This MAY be set if the target platform of this spec is `solaris`.
* **`vm`** (object, OPTIONAL) [Virtual-machine-specific configuration](config-vm.md).
    This MAY be set if the target platform and architecture of this spec support hardware virtualization.

### Example (Linux)

```json
{
    "linux": {
        "namespaces": [
            {
                "type": "pid"
            }
        ]
    }
}
```

## <a name="configHooks" />POSIX-platform Hooks POSIX平台钩子

对于POSIX平台, 配置结构支持 `hooks`，用于配置自定义的与容器生命周期相关的动作[lifecycle](runtime.md#lifecycle).

* **`hooks`** (object, OPTIONAL) MAY 包含如下的属性:
    * **`prestart`** (array of objects, OPTIONAL, **DEPRECATED**) 是一组 [`prestart` hooks](#prestart)钩子.
        * Entries in the array contain the following properties 数组项中包含下列属性:
            * **`path`** (string, REQUIRED) 与 [IEEE Std 1003.1-2008 `execv`'s *path*][ieee-1003.1-2008-functions-exec]语义相似.
                这个规范继承了 IEEE 标准， **`path`** MUST 必须是绝对路径.
            * **`args`** (array of strings, OPTIONAL) 和 [IEEE Std 1003.1-2008 `execv`'s *argv*][ieee-1003.1-2008-functions-exec]标准具有相似语义.
            * **`env`** (array of strings, OPTIONAL) 和 [IEEE Std 1003.1-2008's `environ`][ieee-1003.1-2008-xbd-c8.1]标准具有相似语义.
            * **`timeout`** (int, OPTIONAL) 在钩子执行终止前的秒数.
                一旦设置, `timeout` MUST 一定要大于0.
        * `path`值 MUST 依据 [runtime namespace](glossary.md#runtime-namespace)能够解析.
        * `prestart` 钩子 MUST 能在 [runtime namespace](glossary.md#runtime-namespace)执行.
    * **`createRuntime`** (array of objects, OPTIONAL) 一组创建运行时钩子 [`createRuntime` hooks](#createRuntime-hooks).
        * 数组中的实体包含下述的属性 (和过时的`prestart`具有相同的属性):
            * **`path`** (string, REQUIRED) 与 [IEEE Std 1003.1-2008 `execv`'s *path*][ieee-1003.1-2008-functions-exec]语义相似.
                这个规范继承了 IEEE 标准， **`path`** MUST 必须是绝对路径.
            * **`args`** (array of strings, OPTIONAL) 和 [IEEE Std 1003.1-2008 `execv`'s *argv*][ieee-1003.1-2008-functions-exec]标准具有相似语义.
            * **`env`** (array of strings, OPTIONAL) 和 [IEEE Std 1003.1-2008's `environ`][ieee-1003.1-2008-xbd-c8.1]标准具有相似语义.
            * **`timeout`** (int, OPTIONAL) 在钩子执行终止前的秒数.
                一旦设置, `timeout` MUST 一定要大于0.
        * path`值 MUST 依据 [runtime namespace](glossary.md#runtime-namespace)能够解析.
        * `createRuntime` 钩子 MUST 能在 [runtime namespace](glossary.md#runtime-namespace)执行.
    * **`createContainer`** (array of objects, OPTIONAL) is an array of [`createContainer` hooks](#createContainer-hooks).
        * Entries in the array have the same schema as `createRuntime` entries，与createRuntime相同.
        * The value of `path` MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
        * The `createContainer` hooks MUST be executed in the [container namespace](glossary.md#container-namespace).
    * **`startContainer`** (array of objects, OPTIONAL) is an array of [`startContainer` hooks](#startContainer-hooks).
        * Entries in the array have the same schema as `createRuntime` entries.与createRuntime相同
        * The value of `path` MUST resolve in the [container namespace](glossary.md#container-namespace).
        * The `startContainer` hooks MUST be executed in the [container namespace](glossary.md#container-namespace).
    * **`poststart`** (array of objects, OPTIONAL) is an array of [`poststart` hooks](#poststart).
        * Entries in the array have the same schema as `createRuntime` entries.与createRuntime相同
        * The value of `path` MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
        * The `poststart` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).
    * **`poststop`** (array of objects, OPTIONAL) is an array of [`poststop` hooks](#poststop).
        * Entries in the array have the same schema as `createRuntime` entries.与createRuntime相同
        * The value of `path` MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
        * The `poststop` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).

钩子允许用户在各种生命周期事件前后执行特定的程序.

调用钩子MUST 必须以下列顺序进行.

容器的状态 [state](runtime.md#state)  MUST 必须在标准输入流上传递给钩子，这样他们才能根据当前容器的状态做出合适的工作.

### <a name="configHooksPrestart" />Prestart

The `prestart` hooks MUST be called after the [`start`](runtime.md#start) operation is called but [before the user-specified program command is executed](runtime.md#lifecycle).

在标准方法`start`之后，在用户定义的命令行程序执行之前.

On Linux, for example, they are called after the container namespaces are created, so they provide an opportunity to customize the container (e.g. the network namespace could be specified in this hook).

Linux上，在容器的命名空间创建之前调用，这样才能提供一个自动定义容器的机会.比如，网络命名空间可以用这个钩子来指定。

Note: `prestart` hooks were deprecated in favor of `createRuntime`, `createContainer` and `startContainer` hooks, which allow more granular hook control during the create and start phase.prestart 

钩子已经过时，更乐于推荐`createRuntime`,`createContainer`和`startContainer`钩子，这些钩子可以在创建和启动的prestart相位上提供更细粒度的控制

The `prestart` hooks' path MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
The `prestart` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).

### <a name="configHooksCreateRuntime" />CreateRuntime Hooks

The `createRuntime` hooks MUST be called as part of the [`create`](runtime.md#create) operation after the runtime environment has been created (according to the configuration in config.json) but before the `pivot_root` or any equivalent operation has been executed.作为create标准操作的一部分，在运行时环境变量被创建之后调用。

The `createRuntime` hooks' path MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
The `createRuntime` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).

On Linux, for example, they are called after the container namespaces are created, so they provide an opportunity to customize the container (e.g. the network namespace could be specified in this hook).

The definition of `createRuntime` hooks is currently underspecified and hooks authors, should only expect from the runtime that the mount namespace have been created and the mount operations performed. Other operations such as cgroups and SELinux/AppArmor labels might not have been performed by the runtime.

定义规范不足，命名空间已经绑定完成，其他的cgroup和SELinux/AppArmor标签还没有被运行时执行.

Note: `runc` originally implemented `prestart` hooks contrary to the spec, namely as part of the `create` operation (instead of during the `start` operation). This incorrect implementation actually corresponds to `createRuntime` hooks. For runtimes that implement the deprecated `prestart` hooks as `createRuntime` hooks, `createRuntime` hooks MUST be called after the `prestart` hooks.

runc原始实现了`prestart`钩子，命名作为create操作的一部分，与其命名相反，在start阶段执行操作。

这种错误的实现事实上与`createRuntime`钩子对应，对于实现了过时的`prestart`钩子的运行时，`createRuntime`钩子必须在`prestart`之后调用

### <a name="configHooksCreateContainer" />CreateContainer Hooks 

The `createContainer` hooks MUST be called as part of the [`create`](runtime.md#create) operation after the runtime environment has been created (according to the configuration in config.json) but before the `pivot_root` or any equivalent operation has been executed.

在运行时创建了环境变量后，必须作为`create`一部分，但是在 `pivot_root`或者任何其他的同等操作执行之后

The `createContainer` hooks MUST be called after the `createRuntime` hooks.

必须在`createRuntime`之后

The `createContainer` hooks' path MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
The `createContainer` hooks MUST be executed in the [container namespace](glossary.md#container-namespace).

For example, on Linux this would happen before the `pivot_root` operation is executed but after the mount namespace was created and setup.

pivot_root之前，在命名空间绑定之后

The definition of `createContainer` hooks is currently underspecified and hooks authors, should only expect from the runtime that the mount namespace and different mounts will be setup. Other operations such as cgroups and SELinux/AppArmor labels might not have been performed by the runtime.

定义规范不足，命名空间已经绑定完成，其他的cgroup和SELinux/AppArmor标签还没有被运行时执行.

### <a name="configHooksStartContainer" />StartContainer Hooks

The `startContainer` hooks MUST be called [before the user-specified process is executed](runtime.md#lifecycle) as part of the [`start`](runtime.md#start) operation.
This hook can be used to execute some operations in the container, for example running the `ldconfig` binary on linux before the container process is spawned.

这个钩子被用来在容器里执行一些操作，比如在linux上容器产生之前运行`ldconfig`二进制

The `startContainer` hooks' path MUST resolve in the [container namespace](glossary.md#container-namespace).
The `startContainer` hooks MUST be executed in the [container namespace](glossary.md#container-namespace).

### <a name="configHooksPoststart" />Poststart

The `poststart` hooks MUST be called [after the user-specified process is executed](runtime.md#lifecycle) but before the [`start`](runtime.md#start) operation returns.

在start操作返回之前，在用户定义的进程被拉起后

For example, this hook can notify the user that the container process is spawned.

通知用户容器进程产生了

The `poststart` hooks' path MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
The `poststart` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).

### <a name="configHooksPoststop" />Poststop

The `poststop` hooks MUST be called [after the container is deleted](runtime.md#lifecycle) but before the [`delete`](runtime.md#delete) operation returns.
Cleanup or debugging functions are examples of such a hook.

在容器删除之后，在delete操作返回前

The `poststop` hooks' path MUST resolve in the [runtime namespace](glossary.md#runtime-namespace).
The `poststop` hooks MUST be executed in the [runtime namespace](glossary.md#runtime-namespace).

### Summary 总结

See the below table for a summary of hooks and when they are called:

| Name  | Namespace    |                                                         When                                                                    |
| ---------------------| ------------ | -----------------------------------------------------------------------------------------------------------------|
| `prestart` ( 过时的 )    | runtime   | 在start操作之后调用，在用户定义的命令行程序执行之前                                                                      |
| `createRuntime`         | runtime   | create操作期间,在创建运行时环境之后，在pivot root或者其他的等效操作之前                                                    |
| `createContainer`       | container | create操作期间,在创建运行时环境之后，在pivot root或者其他的等效操作之前                                                    |
| `startContainer`        | container | 在start操作之后调用，在用户定义的命令行程序执行之前                                                                       |
| `poststart`             | runtime   | 在用户定义的进程执行之后，在start操作返回之前                                                                            |
| `poststop`              | runtime   | 在删除容器操作发出之后，但在删除操作执行返回之前                                                                          |

### Example

```json
"hooks": {
    "prestart": [
        {
            "path": "/usr/bin/fix-mounts",
            "args": ["fix-mounts", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        },
        {
            "path": "/usr/bin/setup-network"
        }
    ],
    "createRuntime": [
        {
            "path": "/usr/bin/fix-mounts",
            "args": ["fix-mounts", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        },
        {
            "path": "/usr/bin/setup-network"
        }
    ],
    "createContainer": [
        {
            "path": "/usr/bin/mount-hook",
            "args": ["-mount", "arg1", "arg2"],
            "env":  [ "key1=value1"]
        }
    ],
    "startContainer": [
        {
            "path": "/usr/bin/refresh-ldcache"
        }
    ],
    "poststart": [
        {
            "path": "/usr/bin/notify-start",
            "timeout": 5
        }
    ],
    "poststop": [
        {
            "path": "/usr/sbin/cleanup.sh",
            "args": ["cleanup.sh", "-f"]
        }
    ]
}
```
## <a name="configAnnotations" />Annotations 注解

**`annotations`** (object, OPTIONAL) 包含容器任意的元数据.
This information MAY be structured or unstructured.
这些信息可能是结构化的或者非结构化的
Annotations MUST be a key-value map.
注解必须是一个键值对映射表
If there are no annotations then this property MAY either be absent or an empty map.
如果没有主机，这个属性可以不存在或者是一个空的映射表
Keys MUST be strings.
键必须是字符串
Keys MUST NOT be an empty string.
键必须不是空字符串
Keys SHOULD be named using a reverse domain notation - e.g. `com.example.myKey`.
键应该使用反向的域名符号表示
Keys using the `org.opencontainers` namespace are reserved and MUST NOT be used by subsequent specifications.
`org.opencontainers`是保留的，不能使用这个作为键
Runtimes MUST handle unknown annotation keys like any other [unknown property](#extensibility).
运行时必须处理unknown的键，比如扩展中描述的unknown 属性

Values MUST be strings.
值必须是字符串
Values MAY be an empty string.
值可以是一个空的字符串

```json
"annotations": {
    "com.example.gpu-cores": "2"
}
```

## <a name="configExtensibility" />Extensibility 扩展性

Runtimes MAY [log](runtime.md#warnings) unknown properties but MUST otherwise ignore them.
运行时不知道的属性日志输出告警，但是必须以其他的方式忽略它们
That includes not [generating errors](runtime.md#errors) if they encounter an unknown property.
如果遇到一个unknown的属性，不产生运行时错误

## Valid values 有效值

Runtimes MUST generate an error when invalid or unsupported values are encountered.
遇到不支持的或者无效值，运行时必须产生一个错误
Unless support for a valid value is explicitly required, runtimes MAY choose which subset of the valid values it will support.
除非显式一个有效值支持的需要，运行时可以选自它支持的有效值的集合

## Configuration Schema Example 

Here is a full example `config.json` for reference.

```json
{
    "ociVersion": "1.0.1",
    "process": {
        "terminal": true,
        "user": {
            "uid": 1,
            "gid": 1,
            "additionalGids": [
                5,
                6
            ]
        },
        "args": [
            "sh"
        ],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        "capabilities": {
            "bounding": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "permitted": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "inheritable": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL",
                "CAP_NET_BIND_SERVICE"
            ],
            "effective": [
                "CAP_AUDIT_WRITE",
                "CAP_KILL"
            ],
            "ambient": [
                "CAP_NET_BIND_SERVICE"
            ]
        },
        "rlimits": [
            {
                "type": "RLIMIT_CORE",
                "hard": 1024,
                "soft": 1024
            },
            {
                "type": "RLIMIT_NOFILE",
                "hard": 1024,
                "soft": 1024
            }
        ],
        "apparmorProfile": "acme_secure_profile",
        "oomScoreAdj": 100,
        "selinuxLabel": "system_u:system_r:svirt_lxc_net_t:s0:c124,c675",
        "noNewPrivileges": true
    },
    "root": {
        "path": "rootfs",
        "readonly": true
    },
    "hostname": "slartibartfast",
    "mounts": [
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc"
        },
        {
            "destination": "/dev",
            "type": "tmpfs",
            "source": "tmpfs",
            "options": [
                "nosuid",
                "strictatime",
                "mode=755",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/pts",
            "type": "devpts",
            "source": "devpts",
            "options": [
                "nosuid",
                "noexec",
                "newinstance",
                "ptmxmode=0666",
                "mode=0620",
                "gid=5"
            ]
        },
        {
            "destination": "/dev/shm",
            "type": "tmpfs",
            "source": "shm",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "mode=1777",
                "size=65536k"
            ]
        },
        {
            "destination": "/dev/mqueue",
            "type": "mqueue",
            "source": "mqueue",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys",
            "type": "sysfs",
            "source": "sysfs",
            "options": [
                "nosuid",
                "noexec",
                "nodev"
            ]
        },
        {
            "destination": "/sys/fs/cgroup",
            "type": "cgroup",
            "source": "cgroup",
            "options": [
                "nosuid",
                "noexec",
                "nodev",
                "relatime",
                "ro"
            ]
        }
    ],
    "hooks": {
        "prestart": [
            {
                "path": "/usr/bin/fix-mounts",
                "args": [
                    "fix-mounts",
                    "arg1",
                    "arg2"
                ],
                "env": [
                    "key1=value1"
                ]
            },
            {
                "path": "/usr/bin/setup-network"
            }
        ],
        "poststart": [
            {
                "path": "/usr/bin/notify-start",
                "timeout": 5
            }
        ],
        "poststop": [
            {
                "path": "/usr/sbin/cleanup.sh",
                "args": [
                    "cleanup.sh",
                    "-f"
                ]
            }
        ]
    },
    "linux": {
        "devices": [
            {
                "path": "/dev/fuse",
                "type": "c",
                "major": 10,
                "minor": 229,
                "fileMode": 438,
                "uid": 0,
                "gid": 0
            },
            {
                "path": "/dev/sda",
                "type": "b",
                "major": 8,
                "minor": 0,
                "fileMode": 432,
                "uid": 0,
                "gid": 0
            }
        ],
        "uidMappings": [
            {
                "containerID": 0,
                "hostID": 1000,
                "size": 32000
            }
        ],
        "gidMappings": [
            {
                "containerID": 0,
                "hostID": 1000,
                "size": 32000
            }
        ],
        "sysctl": {
            "net.ipv4.ip_forward": "1",
            "net.core.somaxconn": "256"
        },
        "cgroupsPath": "/myRuntime/myContainer",
        "resources": {
            "network": {
                "classID": 1048577,
                "priorities": [
                    {
                        "name": "eth0",
                        "priority": 500
                    },
                    {
                        "name": "eth1",
                        "priority": 1000
                    }
                ]
            },
            "pids": {
                "limit": 32771
            },
            "hugepageLimits": [
                {
                    "pageSize": "2MB",
                    "limit": 9223372036854772000
                },
                {
                    "pageSize": "64KB",
                    "limit": 1000000
                }
            ],
            "memory": {
                "limit": 536870912,
                "reservation": 536870912,
                "swap": 536870912,
                "kernel": -1,
                "kernelTCP": -1,
                "swappiness": 0,
                "disableOOMKiller": false
            },
            "cpu": {
                "shares": 1024,
                "quota": 1000000,
                "period": 500000,
                "realtimeRuntime": 950000,
                "realtimePeriod": 1000000,
                "cpus": "2-3",
                "mems": "0-7"
            },
            "devices": [
                {
                    "allow": false,
                    "access": "rwm"
                },
                {
                    "allow": true,
                    "type": "c",
                    "major": 10,
                    "minor": 229,
                    "access": "rw"
                },
                {
                    "allow": true,
                    "type": "b",
                    "major": 8,
                    "minor": 0,
                    "access": "r"
                }
            ],
            "blockIO": {
                "weight": 10,
                "leafWeight": 10,
                "weightDevice": [
                    {
                        "major": 8,
                        "minor": 0,
                        "weight": 500,
                        "leafWeight": 300
                    },
                    {
                        "major": 8,
                        "minor": 16,
                        "weight": 500
                    }
                ],
                "throttleReadBpsDevice": [
                    {
                        "major": 8,
                        "minor": 0,
                        "rate": 600
                    }
                ],
                "throttleWriteIOPSDevice": [
                    {
                        "major": 8,
                        "minor": 16,
                        "rate": 300
                    }
                ]
            }
        },
        "rootfsPropagation": "slave",
        "seccomp": {
            "defaultAction": "SCMP_ACT_ALLOW",
            "architectures": [
                "SCMP_ARCH_X86",
                "SCMP_ARCH_X32"
            ],
            "syscalls": [
                {
                    "names": [
                        "getcwd",
                        "chmod"
                    ],
                    "action": "SCMP_ACT_ERRNO"
                }
            ]
        },
        "namespaces": [
            {
                "type": "pid"
            },
            {
                "type": "network"
            },
            {
                "type": "ipc"
            },
            {
                "type": "uts"
            },
            {
                "type": "mount"
            },
            {
                "type": "user"
            },
            {
                "type": "cgroup"
            }
        ],
        "maskedPaths": [
            "/proc/kcore",
            "/proc/latency_stats",
            "/proc/timer_stats",
            "/proc/sched_debug"
        ],
        "readonlyPaths": [
            "/proc/asound",
            "/proc/bus",
            "/proc/fs",
            "/proc/irq",
            "/proc/sys",
            "/proc/sysrq-trigger"
        ],
        "mountLabel": "system_u:object_r:svirt_sandbox_file_t:s0:c715,c811"
    },
    "annotations": {
        "com.example.key1": "value1",
        "com.example.key2": "value2"
    }
}
```


[apparmor]: https://wiki.ubuntu.com/AppArmor
[cgroup-v1-memory_2]: https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt
[selinux]:http://selinuxproject.org/page/Main_Page
[no-new-privs]: https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt
[proc_2]: https://www.kernel.org/doc/Documentation/filesystems/proc.txt
[umask.2]: http://pubs.opengroup.org/onlinepubs/009695399/functions/umask.html
[semver-v2.0.0]: http://semver.org/spec/v2.0.0.html
[ieee-1003.1-2008-xbd-c8.1]: http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap08.html#tag_08_01
[ieee-1003.1-2008-functions-exec]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html
[naming-a-volume]: https://aka.ms/nb3hqb

[capabilities.7]: http://man7.org/linux/man-pages/man7/capabilities.7.html
[mount.2]: http://man7.org/linux/man-pages/man2/mount.2.html
[mount.8]: http://man7.org/linux/man-pages/man8/mount.8.html
[mount.8-filesystem-independent]: http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-INDEPENDENT_MOUNT_OPTIONS
[mount.8-filesystem-specific]: http://man7.org/linux/man-pages/man8/mount.8.html#FILESYSTEM-SPECIFIC_MOUNT_OPTIONS
[getrlimit.2]: http://man7.org/linux/man-pages/man2/getrlimit.2.html
[getrlimit.3]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/getrlimit.html
[stdin.3]: http://man7.org/linux/man-pages/man3/stdin.3.html
[uts-namespace.7]: http://man7.org/linux/man-pages/man7/namespaces.7.html
[zonecfg.1m]: http://docs.oracle.com/cd/E86824_01/html/E54764/zonecfg-1m.html
