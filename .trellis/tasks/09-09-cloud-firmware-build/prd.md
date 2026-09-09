# 云端固件打包 GitHub Action

## Goal

新增一个 GitHub Actions workflow,用于**云端编译固件并打包成 `.bin`**,按
"PlatformIO 环境关键字"选择要编的构型,产物带版本号,可上传 Artifact 并可选创建 GitHub Release。

- 宿主仓库(触发方)暂未定("静态项目")——脚本需自包含、可平移到任意仓库。
- 默认/示意目标是本固件仓库 `Mlxa0324/qingwa-pnp-marlin` 的 `Opulo_Lumen_REV5` 环境。

## Requirements

- 触发:`workflow_dispatch` 手动(可扩展 `workflow_call` 供宿主矩阵复用)。
- 关键字匹配:输入一个/多个关键字(逗号分隔),匹配 `platformio.ini`(含
  `ini/*.ini` 的 `extra_configs`)里所有 **env 名**,取"名称包含关键字(不区分大小写)"者编译。
  - 为空时退回 `platformio.ini` 的 `default_envs`。
- 产物:
  - 每个 env 编出的 `firmware.{bin,hex,uf2}`(排除 .elf)重命名为
    `<env>-<version>.<ext>`,按 env 归入 `dist/<env>-<version>/`。
  - 打包单个 `dist/固件包-<version>.zip` 方便整包下载。
  - 上传 Actions Artifact(`firmware-<version>`),内含 zip。
- 版本号:
  - `version` 输入,默认留空则自动 `YYYYMMDD-HHMMSS-短SHA`。
- Release(可选):`create_release=true` + `release_tag` 时,用
  `softprops/action-gh-release` 把产物挂到指定 tag(建议 `draft: true`)。
- 环境:自动装 PlatformIO、缓存 `~/.platformio`、装依赖、逐 env 编译失败即红。
- 跨仓库:提供 `source_repo` / `source_ref` 输入,默认构建"运行本 workflow 的仓库自身";
  静态宿主构建本私有固件仓库时,需通过 `secrets.FIRMWARE_PAT` 提供 PAT(脚本里做
  fallback: 有 PAT 用 PAT,否则用 `github.token`)。

## Acceptance Criteria

- [ ] `.github/workflows/build-firmware.yml` 创建,包含 workflow_dispatch + workflow_call 双触发与上述全部 inputs。
- [ ] 构建选择逻辑用单个 Python 脚本实现(发现 env → 关键字过滤 → 逐 env `pio run` → 收集/改名/打 zip)。
- [ ] `source_repo/source_ref` 为空时退化为运行仓库自身;checkout 用 `FIRMWARE_PAT || github.token`。
- [ ] Artifact 上传 + 可选 Release 步骤齐全;YAML 语法校验通过。
- [ ] 文件顶部用中文注释说明用法、各 input 含义、放静态宿主时如何迁移。
- [ ] 任务收尾:`finish-work` 检查后,本任务可 archive(宿主仓库定后平移再启用)。

## Technical Notes

- 环境发现用 `pio project config`(会把 `extra_configs` 合并)解析 `[env:...]` 段。
- buildroot 的 Auto Build/`make tests-single-ci` 是编译入口参考,但本任务直接 `pio run -e <env>`。
- 产物路径约定:`.pio/build/<env>/firmware.{bin,hex,uf2}`。
- 上传 Release 到"被构建的仓库"而非宿主,跨私有仓需要 PAT;注释说明。
