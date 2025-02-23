# 降级 StarRocks

本文介绍如何降级您的 StarRocks 集群。

如果升级 StarRocks 集群后出现异常，您可以将其降级到之前的版本，以快速恢复集群。

## 概述

请在降级前查看本节中的信息。建议您按照文中推荐的操作降级集群。

### 降级路径

- **小版本降级**

  您可以跨小版本降级您的 StarRocks 集群，例如，从 v2.2.11 直接降级到 v2.2.6。

- **大版本降级**

  出于兼容性和安全原因，我们强烈建议您将 StarRocks 集群按**大版本逐级降级**。例如，要将 StarRocks v2.5 集群降级到 v2.2，需要按照以下顺序降级：v2.5.x --> v2.4.x --> v2.3.x --> v2.2.x。

- **重大版本降级**

  - 您无法跨版本降级至 v1.19，必须先降级至 v2.0。
  - 您只能将集群从 v3.0 降级到 v2.5.3 以上版本。
    - StarRocks 在 v3.0 版本中升级了 BDB 库。由于 BDB JE 无法回滚，所以降级后您必须继续使用 v3.0 的 BDB 库。
    - 升级至 v3.0 后，集群默认使用新的 RBAC 权限系统。降级后您只能使用 RBAC 权限系统。

### 降级流程

StarRocks 的降级流程与 [升级流程](../deployment/upgrade.md#升级流程) 相反。所以**您需要先降级 FE，再降级 BE 和CN**。错误的降级顺序可能会导致 FE 与 BE/CN 不兼容，进而导致服务崩溃。对于 FE 节点，您必须先降级所有 Follower FE 节点，最后降级 Leader FE 节点。

## 准备工作

准备过程中，如果您需要进行大版本或重大版本降级，则必须进行兼容性配置。在全面降级集群所有节点之前，您还需要对其中一个 FE 和 BE 节点进行降级正确性测试。

### 兼容性配置

如需进行大版本或重大版本降级，则必须进行兼容性配置。降级前版本不同，兼容性配置也不同。

#### 自 v2.2 及以后版本降级

设置 FE 配置项 `ignore_unknown_log_id` 为 `true`。由于该配置项为静态参数，所以必须在 FE 配置文件 **fe.conf** 中修改，并且在修改完成后重启节点使修改生效。降级结束且第一次 Checkpoint 完成后，您可以将其重置为 `false` 并重新启动节点。

#### 自 v2.4 及以后版本降级

如果您启用了 FQDN 访问，则必须在降级之前切换到 IP 地址访问。有关详细说明，请参考 [回滚 FQDN](../administration/enable_fqdn.md#回滚)。

### 降级正确性测试

在降级生产集群中的所有节点之前，强烈建议您在单个 BE 和 FE 节点上进行降级正确性测试，以查看降级是否影响您当前的数据。

#### FE 降级正确性测试

按照以下步骤执行 FE 降级正确性测试：

1. 在开发环境中部署一个要降级的目标版本的测试 FE 节点。有关详细说明，请参考 [手动部署 StarRocks - 启动 Leader FE 节点](../deployment/deploy_manually.md#第一步启动-leader-fe-节点)。
2. 修改测试 FE 节点的配置文件 **fe.conf**：

   - 为测试 FE 节点设置与生产集群不同的 `http_port`、`rpc_port`、`query_port` 以及 `edit_log_port`。
   - 添加配置项 `cluster_id = 123456`。
   - 添加配置项 `metadata_failure_recovery = true`。

3. 复制生产集群 Leader FE 节点的 **meta** 目录，粘贴至测试 FE 节点的部署目录中。
4. 修改测试 FE 节点的文件 **meta/image/VERSION**。将 `cluster_id` 设置为 `123456`。
5. 启动测试 FE 节点。

   ```Bash
   sh bin/start_fe.sh --daemon
   ```

6. 查看节点是否启动成功。

   ```Bash
   ps aux | grep StarRocksFE
   ```

   - 如果该测试 FE 节点启动成功，则可以安全降级生产集群中的 FE 节点。
   - 如果该测试 FE 节点启动失败，则需要在 FE 日志文件 **fe.log** 中查看失败原因并解决问题。如果问题无法解决，您可以直接删除该节点。

#### BE/CN 降级正确性测试

> **注意**
>
> BE 降级正确性测试将导致丢失一个数据副本。因此在执行测试之前，请确保您至少拥有三个完整的数据副本。

按照以下步骤执行 BE/CN 降级正确性测试：

1. 随机选择一个 BE/CN 节点，进入其工作路径，并停止该节点。

   - BE 节点：

     ```Bash
     # 将 <be_dir> 替换为 BE 节点的部署目录。
     cd <be_dir>/be
     ./bin/stop_be.sh
     ```

   - CN 节点：

     ```Bash
     # 将 <cn_dir> 替换为 CN 节点的部署目录。
     cd <cn_dir>/be
     ./bin/stop_cn.sh
     ```

2. 替换部署文件原有路径 **bin** 和 **lib** 为旧版本的部署文件。

   ```Bash
   mv lib lib.bak 
   mv bin bin.bak
   cp -r /tmp/StarRocks-x.x.x/be/lib  .
   cp -r /tmp/StarRocks-x.x.x/be/bin  .
   ```

3. 启动该 BE/CN 节点。

   - BE 节点：

     ```Bash
     sh bin/start_be.sh --daemon
     ```

   - CN 节点：

     ```Bash
     sh bin/start_cn.sh --daemon
     ```

4. 查看节点是否已启动成功。

   ```Bash
   ps aux | grep starrocks_be
   ```

   - 如果该 BE/CN 节点启动成功，则可以安全降级其他 BE/CN 节点。
   - 如果该 BE/CN 节点启动失败，则需要查看日志文件中的失败原因并解决问题。如果问题无法解决，您可以删除该节点，清理数据，并用先前版本的部署文件重启该节点，最后将该节点重新加入集群。

## 降级 FE

通过降级正确性测试后，您可以先降级 FE 节点。您必须先降级 Follower FE 节点，然后再降级 Leader FE 节点。

1. 进入 FE 节点工作路径，并停止该节点。

   ```Bash
   # 将 <fe_dir> 替换为 FE 节点的部署目录。
   cd <fe_dir>/fe
   ./bin/stop_fe.sh
   ```

2. 替换部署文件原有路径 **bin**、**lib** 以及 **spark-dpp** 为旧版本的部署文件。

   ```Bash
   mv lib lib.bak 
   mv bin bin.bak
   mv spark-dpp spark-dpp.bak
   cp -r /tmp/StarRocks-x.x.x/fe/lib  .   
   cp -r /tmp/StarRocks-x.x.x/fe/bin  .
   cp -r /tmp/StarRocks-x.x.x/fe/spark-dpp  .
   ```

   > **注意**
   >
   > 如需将 StarRocks v3.0 降级至 v2.5，则必须在替换部署文件后执行以下步骤：
   >
   > 1. 将 v3.0 部署文件中的**fe/lib/starrocks-bdb-je-18.3.13.jar** 复制到 v2.5 部署文件的 **fe/lib** 路径下。
   > 2. 删除文件 **fe/lib/je-7.\*.jar**。

3. 启动该 FE 节点。

   ```Bash
   sh bin/start_fe.sh --daemon
   ```

4. 查看节点是否启动成功。

   ```Bash
   ps aux | grep StarRocksFE
   ```

5. 重复以上步骤降级其他 Follower FE 节点，最后降级 Leader FE 节点。

   > **注意**
   >
   > 如需将 StarRocks v3.0 降级至 v2.5，则必须在降级完成后执行以下步骤：
   >
   > 1. 执行 [ALTER SYSTEM CREATE IMAGE](../sql-reference/sql-statements/Administration/ALTER%20SYSTEM.md) 创建新的元数据快照文件。
   > 2. 等待元数据快照文件同步至其他 FE 节点。
   >
   > 如果不运行该命令，部分降级操作可能会失败。ALTER SYSTEM CREATE IMAGE 命令仅在 v2.5.3 及更高版本支持。

## 降级 BE

降级所有 FE 节点后，您可以继续降级 BE 节点。

1. 进入 BE 节点工作路径，并停止该节点。

   ```Bash
   # 将 <be_dir> 替换为 BE 节点的部署目录。
   cd <be_dir>/be
   ./bin/stop_be.sh
   ```

2. 替换部署文件原有路径 **bin** 和 **lib** 为旧版本的部署文件。

   ```Bash
   mv lib lib.bak 
   mv bin bin.bak
   cp -r /tmp/StarRocks-x.x.x/be/lib  .
   cp -r /tmp/StarRocks-x.x.x/be/bin  .
   ```

3. 启动该 BE 节点。

   ```Bash
   sh bin/start_be.sh --daemon
   ```

4. 查看节点是否启动成功。

   ```Bash
   ps aux | grep starrocks_be
   ```

5. 重复以上步骤降级其他 BE 节点。

## 降级 CN

1. 进入 CN 节点工作路径，并优雅停止该节点。

   ```Bash
   # 将 <cn_dir> 替换为 CN 节点的部署目录。
   cd <cn_dir>/be
   ./bin/stop_cn.sh --graceful
   ```

2. 替换部署文件原有路径 **bin** 和 **lib** 为新版本的部署文件。

   ```Bash
   mv lib lib.bak 
   mv bin bin.bak
   cp -r /tmp/StarRocks-x.x.x/be/lib  .
   cp -r /tmp/StarRocks-x.x.x/be/bin  .
   ```

3. 启动该 CN 节点。

   ```Bash
   sh bin/start_cn.sh --daemon
   ```

4. 查看节点是否启动成功。

   ```Bash
   ps aux | grep starrocks_be
   ```

5. 重复以上步骤降级其他 CN 节点。
