# Overview 🔥🔥🔥
---
**Lustre:** 基于GNU GPL协议开源的分布式并行文件系统。由于其容易扩展和极致的性能，常被用在超算、AI领域、视频存储等领域。可以同时支持数百PB的数据存储和每秒数百GB的聚合吞吐量。

# Build 🤖
---
**注意：** 该仓库适配了特定的内核源代码版本；
* **Lustre版本:** **2.15.57**
* **内核版本：** **5.10.0-60.18.0.50.oe2203.x86_64**


**构建rpm包:**

    ./auto_build_rpms.sh --with-linux /usr/src/linux-5.10.0-60.18.0.50.oe2203.x86_64/

**--with-linux:** 指定内核源码路径

**输出：** **lustre-package-2.15.57.tar.gz**

# Installation 📦
---

将**lustre-package-2.15.57.tar.gz**压缩包通过scp命令拷贝到目标机器上，解压得到**lustre-package-2.15.57**目录。

 **1、修改配置文件**

在开始安装之前，需要根据实际情况修改lustreserver.yaml配置文件。
```yaml
lustre:
  lustre_dir: /home/w2938/demo  # Lustre安装目录,需替换实际路径
  fsname: lustrefs              # Lustre文件系统名称，需替换
  client_index: 0               # 客户端节点下标，需替换
  client_dir: lustre            # Lustre挂载目录，需替换
  stride: 32                    # 后续使用，暂不用设置
  ost_nums: 12                  # OST数量
  nodes:
    - name: node86              # 节点1名称，需替换
      ip: 192.168.18.14         # 节点1IP，需替换
      net: bond0                # 节点1网络接口名称，需替换
      roles: ["mgt", "mdt", "ost"]
      lun:                      # 节点1LUN配置，需替换
        mgt: /dev/disk/by-id/dm-uuid-mpath-3600b342ace61b9d684645831ee00007c
        mdt: /dev/disk/by-id/dm-uuid-mpath-3600b342ace61b9d67ff1ca14d400006f
        ost:
          - /dev/disk/by-id/dm-uuid-mpath-3600b342ace61b9d67ff1c8adcb00006d
    - name: node1               # 节点2名称，需替换
      ip: 192.168.18.15         # 节点2IP，需替换
      net: bond0                # 节点2网络接口名称，需替换
      roles: ["mdt", "ost"]      
      lun:                      # 节点2LUN配置，需替换
        mdt: /dev/disk/by-id/dm-uuid-mpath-3600b342ace61b9d67ff1c9fb6800006e
        ost:
          - /dev/disk/by-id/dm-uuid-mpath-3600b342ace61b9d67ff1c542d700006c
```
**2、多节点批量操作脚本**

  通过SSH在多节点集群上批量执行Lustre文件系统的全生命周期管理，支持原子化操作和组合操作。
**注意：** 执行脚本之前，多个节点需要配置**免密**操作

     ssh-copy-id root@目标ip

**lustre_exec.sh 参数说明**
| 阶段参数      | 操作描述                                   | 关联单节点命令                |
|---------------|------------------------------------------|-----------------------------|
| `--stage install`   | 在所有节点安装Lustre RPM包                | `./auto_install.sh`            |
| `--stage format`    | 配置格式化Lustre存储设备      | `./auto_format.sh -i index`    |
| `--stage mount`     | 挂载Lustre文件系统到配置的挂载点          | `./auto_mount.sh -i index`     |
| `--stage unmount`   | 卸载所有节点上的Lustre文件系统            | `./auto_unmount.sh -i index`   |
| `--stage uninstall` | 卸载所有节点的Lustre软件包                | `./auto_uninstall.sh`          |
| `--stage start`     | 组合操作：`install` → `format` → `mount`  | 按顺序执行三个阶段          |
| `--stage end`       | 组合操作：`unmount` → `uninstall`         | 按顺序执行两个阶段          |

**使用示例**
```bash
# 完整部署流程（安装->格式化->挂载）
./lustre_exec.sh --stage start

# 仅执行安装操作
./lustre_exec.sh --stage install

# 销毁环境（卸载->删除软件）
./lustre_exec.sh --stage end
```

**3、单节点操作脚本**

**单节点操作说明** 

| 命令                     | 操作描述                          | 参数说明                                    |
|--------------------------|----------------------------------|---------------------------------------------|
| `./auto_install.sh`         | 单节点安装 Lustre 软件包          | 无参数                                      |
| `./auto_format.sh -i index` | 单节点格式化 Lustre 文件系统       | `-i index`: 节点下标（从0开始，必需参数）    |
| `./auto_mount_mgt.sh -i index`  | 挂载MGT（MGT只在主节点上挂载，非主节点跳过）                 | `-i index`: 节点下标（从0开始，必需参数）  |
| `./auto_mount_mdt.sh -i index`  | 单节点挂载MDT                 | `-i index`: 节点下标（从0开始，必需参数）  |
| `./auto_mount_ost.sh -i index`| 单节点挂载OST        | `-i index`: 节点下标（从0开始，必需参数）    |
| `./auto_mount_client.sh`       | 单节点挂载客户端           | 无参数                                      |
| `./auto_mount.sh -i index`  | 单节点依次挂载MGT、OST以及客户端        | `-i index`: 节点下标（从0开始，必需参数）    |
| `./auto_umount_mgt.sh -i index`  | 卸载MGT(非主节点跳过)                 | `-i index`: 节点下标（从0开始，必需参数）  |
| `./auto_umount_mdt.sh -i index`  | 单节点卸载MDT                 | `-i index`: 节点下标（从0开始，必需参数）  |
| `./auto_umount_ost.sh -i index`| 单节点卸载OST        | `-i index`: 节点下标（从0开始，必需参数）    |
| `./auto_umount_client.sh`       | 单节点卸载客户端           | 无参数                                      |
| `./auto_umount.sh -i index`| 单节点依次卸载客户端、OST以及客户端        | `-i index`: 节点下标（从0开始，必需参数）    |
| `./auto_uninstall.sh`       | 单节点卸载 Lustre 软件包           | 无参数                                      |

**使用示例**
```bash
# 单节点安装
./auto_install.sh

# 单节点格式化
./auto_format.sh -i 0

# 单节点挂载
./auto_mount.sh -i 0

# 单节点卸载
./auto_umount.sh -i 0

# 单节点卸载lustre
./auto_uninstall.sh
```