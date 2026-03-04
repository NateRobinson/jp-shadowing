# /gen-lesson — 日语影子跟读练习页面生成器

用户会告诉你**页码**和**音频文件名**。按以下步骤生成一课的练习页面。

## 命名规范

目录使用 `L{课号}-{本文号}` 格式，如 `L01-1`（第1課 本文1）、`L01-2`（第1課 本文2）。
课号和本文号从 PDF 内容中识别。

## 工作流程

### 1. 从 PDF 提取对话内容

用 `pdftoppm` 将指定页转为图片：

```bash
# 检查缓存（避免重复转换）
ls pdf-pages/page-$(printf '%03d' {PAGE_NUM}).png 2>/dev/null || \
  (mkdir -p pdf-pages && pdftoppm -png -f {PAGE_NUM} -l {PAGE_NUM} -r 300 ../N5电子版教材.pdf pdf-pages/page)
```

其中 `{PAGE_NUM}` 是用户提供的页码。PDF 文件在项目父目录下（`../N5电子版教材.pdf`）。

然后用 Read 工具读取生成的图片，提取：
- 课号（第X課）
- 课文标题
- 场景说明（括号内）
- 对话内容（说话人 + 台词）
- 可提炼的重点句型

### 2. 日文 OCR 注意事项

从图片提取日文时务必注意：
- 区分助词 **は**（读 wa）与 **わ**
- 区分助词 **を** 与 **お**
- 注意小假名：**っ ゃ ゅ ょ** 与大写版本
- 长音符号 **ー** 不要误认为横线或破折号
- 汉字和假名的正确对应（如 **来ました** 不是 **来ました**）
- **。** 是句号，**、** 是逗号，不要遗漏

### 3. 创建目录结构

```bash
mkdir -p L{LESSON_NUM}-{HONBUN_NUM}
cp ../mp3/{AUDIO_FILE} L{LESSON_NUM}-{HONBUN_NUM}/
```

其中 `{LESSON_NUM}` 是两位数课号（如 01），`{HONBUN_NUM}` 是本文序号（如 1、2）。

### 4. 生成 対話.md

创建 `L{LESSON_NUM}-{HONBUN_NUM}/対話.md`，结构如下：

```markdown
# 第X課 课文标题

> 教材：文化初級日本語 I（テキスト 改訂版）
> ページ：{PAGE_NUM}
> 音声：{AUDIO_ID}（{AUDIO_FILE}）

## 本文 1

**（场景说明）**

**说话人A：** 日文台词

**说话人B：** 日文台词

...

## 中文翻译

**（场景说明中文）**

**中文名A：** 中文翻译

**中文名B：** 中文翻译

...

## 重点句型

| 句型 | 含义 |
|------|------|
| 句型1 | 含义1 |
| 句型2 | 含义2 |
```

**翻译指南：**
- 使用自然的中文，不要逐字直译
- 人名转换：日文片假名人名 → 中文音译（如 ワン → 王、ラフル → 拉弗尔）
- 场景说明也要翻译

**N5 句型分析指南：**
- 提取课文中出现的核心语法点
- 常见 N5 句型：～は～です、～から来ました、～ではありません、～ですか、～の～、～を～ます、～に行きます 等
- 每个句型给出简明中文释义

### 5. 生成 index.html

**务必先读取 `L01-1/index.html` 作为模板**，然后只替换内容区域，保留完全一致的 CSS 和 JS。

具体做法：用 Read 工具读取 `L01-1/index.html`，然后基于其完整内容生成新页面，只修改以下部分：

**需要替换的变量：**
- `<title>` → `第X課 课文标题`
- `.lesson-tag` 文本 → `第X課`
- `.lesson-title` 文本 → 课文标题
- `.lesson-meta` 文本 → `文化初級日本語 I // p.{PAGE_NUM} // track {AUDIO_ID}`
- `STORAGE_KEY` 常量 → `jp-study-plays-{AUDIO_FILE}`
- `<audio src>` → `./AUDIO_FILE`

**需要替换的内容区域：**
- `.scene-note` → 场景说明（日文原文）
- `.dialogue-line` 列表 → 每行需加 `data-idx="0"` 递增属性: `.speaker`（片假名名字的英文大写）+ `.jp-text`（日文 + ruby 注音）+ `.cn-text`（中文翻译）
- `.grammar-card` 列表 → 每张: `.grammar-jp`（句型 + ruby 注音）+ `.grammar-cn`（中文释义）

**Ruby 注音规则（振り仮名）：**
- 只在汉字上加 ruby，假名不加
- 格式：`<ruby>漢字<rt>かんじ</rt></ruby>`
- 例：`<ruby>私<rt>わたし</rt></ruby>はワン・シューミンです。`

**Speaker 命名规则：**
- 片假名人名转英文大写：ワン → WAN、ラフル → RAHUL、たなか → TANAKA
- 如果是职业/身份：先生 → SENSEI、店員 → CLERK

**需要替换的 JS 变量：**
- `ts` 数组 → 每行对话的音频起始时间戳（见步骤 5.1）

**绝对不能改动的部分：**
- 所有 CSS（`:root` 变量、`[data-theme="dark"]`、所有类定义、sync highlight、sticky player）
- 所有 JS 逻辑（主题切换、播放器、A-B 复读、拖拽进度条、速度控制、播放计数、振り仮名/中文 toggle、localStorage 持久化、auto-highlight + click-to-seek、键盘空格控制）
- HTML 结构骨架（`.container` > `.lesson-header` > `.player` > `.content-toggles` > `.section` DIALOGUE > `.section` GRAMMAR > `.footer`）

### 5.1 生成音频时间戳

用 ffmpeg 分析音频中的静音间隔，确定每行对话的起始时间：

```bash
ffmpeg -i L{LESSON_NUM}-{HONBUN_NUM}/{AUDIO_FILE} -af silencedetect=noise=-25dB:d=0.3 -f null - 2>&1 | grep -E "silence_(start|end)"
```

根据静音间隔（通常 ≥0.8s 为说话人切换）映射到对话行，得到每行的起始秒数数组。将结果填入 JS 中的 `ts` 数组，如 `ts=[4.7,10.9,15.6,17.5]`。

### 6. 更新 README

在 `README.md` 的对应课程表格中追加新行。如果是新课（新的第X課），先添加课程标题行 `### 第X課`，再添加表格。

### 7. 发布到 MyVibe

使用 `/myvibe-publish --dir L{LESSON_NUM}-{HONBUN_NUM} --new` 发布。

**Vibe 标题命名规则：** `JP Shadow Reading Lesson {序号}`，序号为两位数递增（01, 02, 03...），按课文生成的先后顺序编号，不按课号或本文号。

发布后将生成的 MyVibe 链接更新到 README 课程表格中。

### 8. 汇报结果

完成后输出：
- 列出生成的文件
- 提示用户预览：`open L{LESSON_NUM}-{HONBUN_NUM}/index.html`
- 附上 MyVibe 在线链接
