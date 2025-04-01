# pdfplumber

[![Version](https://img.shields.io/pypi/v/pdfplumber.svg)](https://pypi.python.org/pypi/pdfplumber) ![Tests](https://github.com/jsvine/pdfplumber/workflows/Tests/badge.svg) [![Code coverage](https://codecov.io/gh/jsvine/pdfplumber/branch/stable/graph/badge.svg)](https://codecov.io/gh/jsvine/pdfplumber/branch/stable) [![Support Python versions](https://img.shields.io/pypi/pyversions/pdfplumber.svg)](https://pypi.python.org/pypi/pdfplumber)

解析 PDF 文件,获取相关的文本字符、矩形和线条的详细信息。 额外功能：表格提取和可视化调试。

用于电脑生成的 PDF 上效果最好，不支持扫描的 PDF，基于[`pdfminer.six`](https://github.com/goulu/pdfminer)实现.

当前版本 [测试用例](tests/) 已经在 [Python 3.8, 3.9, 3.10, 3.11](.github/workflows/tests.yml) 验证通过.

**反馈 BUG** 或提交功能, 请 [提交 issue](https://github.com/jsvine/pdfplumber/issues/new/choose). **有疑问** 或提交需要协助的特殊 PDF 文件, 请 [进入讨论区](https://github.com/jsvine/pdfplumber/discussions).

> 👋 本项目维护人员提供 PDF 数据提取服务. 询价可联系 [Jeremy](https://www.jsvine.com/consulting/pdf-data-extraction/) (针对任何项目) 或 [Samkit](https://www.linkedin.com/in/samkit-jain/) (针对表格提取).

## 目录

- [安装](#installation)
- [命令行](#command-line-interface)
- [Python 包](#python-library)
- [可视化调试](#visual-debugging)
- [提取文本](#extracting-text)
- [提取表格](#extracting-tables)
- [提取表单域值](#extracting-form-values)
- [用例演示](#demonstrations)
- [与其他库比较](#comparison-to-other-libraries)
- [认可及贡献](#acknowledgments--contributors)
- [贡献代码](#contributing)

## 安装

```sh
pip install pdfplumber
```

## 命令行

### 基础示例

```sh
curl "https://raw.githubusercontent.com/jsvine/pdfplumber/stable/examples/pdfs/background-checks.pdf" > background-checks.pdf
pdfplumber < background-checks.pdf > background-checks.csv
```

输出将是一个 CSV，包含 PDF 中每个字符、行和矩形的信息。

### 配置项

| 参数                         | 说明                                                                                                     |
| ---------------------------- | -------------------------------------------------------------------------------------------------------- |
| `--format [输出格式]`        | `csv` 或 `json`. `json` 返回信息包括 PDF 文件级和页面级元数据，以及字典嵌套属性。                        |
| `--pages [页码]`             | `1`-页面索引或页面范围. 举例: `1, 11-15`, 指定的页面范围: 1, 11, 12, 13, 14, 15.                         |
| `--types [提取对象类型列表]` | 可选值: `char`, `rect`, `line`, `curve`, `image`, `annot`等,默认全部提取                                 |
| `--laparams`                 | json 格式字符串 (例: `'{"detect_vertical": true}'`) PDF 打开的传参 `pdfplumber.open(..., laparams=...)`. |
| `--precision [整数]`         | 浮点数四舍五入的小数位数。默认为无舍入。                                                                 |

## Python 库

### 基础示例

```python
import pdfplumber

with pdfplumber.open("path/to/file.pdf") as pdf:
    first_page = pdf.pages[0]
    print(first_page.chars[0])
```

### 加载 PDF

调用`pdfplumber.open(x)`加载 PDF, 其中`x`可以有以下几种格式:

- PDF 文件路径

- 文件对象, 以字节流形式加载

- 类文件对象, 以字节流形式加载

`open`方法加载文件后返回一个 `pdfplumber.PDF` 实例.

传入 `password` 参数用于加载已加密的 PDF 文件, 例: `pdfplumber.open("file.pdf", password = "test")`.

传入 `laparams` 参数可以使用`pdfminer.six`的布局引擎用于布局分析, 例: `pdfplumber.open("file.pdf", laparams = { "line_overlap": 0.7 })`.

Invalid metadata values are treated as a warning by default. If that is not intended, pass `strict_metadata=True` to the `open` method and `pdfplumber.open` will raise an exception if it is unable to parse the metadata.

默认情况下，无效元数据值只是被视为警告。如果设置`strict_metadata=True`。当加载 PDF 后无法分析元数据，将抛出异常。

### `pdfplumber.PDF` 类

`pdfplumber.PDF` 类代表一个 PDF 文件,主要有以下两个属性:

| 属性        | 说明                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------- |
| `.metadata` | 元数据键/值对字典，摘自 PDF 的“信息”。通常包括“CreationDate”(创建日期)、“ModDate”(修改日期)、“Producer”(创建者)等。 |
| `.pages`    | 包含`pdfplumber.Page`(页实例)的列表。                                                                               |

... 包含以下方法:

| 方法       | 说明                                                                                                                         |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `.close()` | 调用此方法会在每个页面上调用 `Page.close()`，同时也会关闭文件流（除非文件流是外部的，即已经打开并直接传递给 `pdfplumber`）。 |

### `pdfplumber.Page` 类

`pdfplumber.Page`是`pdfplumber`核心. 大部分的操作都是围绕此类进行.主要包含以下属性:

| 属性                                                                | 说明                                                                                                                            |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `.page_number`                                                      | 页码, `1`第一页, `2`第二页, 以此类推.                                                                                           |
| `.width`                                                            | 页面宽.                                                                                                                         |
| `.height`                                                           | 页面高.                                                                                                                         |
| `.objects` / `.chars` / `.lines` / `.rects` / `.curves` / `.images` | 这些属性中的每一个都是一个列表，每个列表都为嵌入在页面上的每个此类对象包含一个字典。有关详细信息，请参见 "[Objects](#objects)". |

... `pdfplumber.Page`的主要方法:

| 方法                                                       | 说明                                                                                                                                                                                                                                                                                                                                                                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.crop(bounding_box, relative=False, strict=True)`         | 返回裁剪边界框中的页面内容，边界框应表示为 4 元组，值为 `(x0, top, x1, bottom)`。裁剪的页面保留至少部分位于边界框内的对象。如果对象仅部分落在长方体中，则将剖切其被边界框包含的部分。如果`relative=True`，则边界框计算为距页面左上角的偏移量，而不是绝对位置。 (参考 [Issue #245](https://github.com/jsvine/pdfplumber/issues/245) 给出的直观示例和说明.) 当`strict=True` (默认值), 裁剪框必须在页面边界内部. |
| `.within_bbox(bounding_box, relative=False, strict=True)`  | 类似`.crop`,但仅保留完全落在边界框内的对象。                                                                                                                                                                                                                                                                                                                                                                  |
| `.outside_bbox(bounding_box, relative=False, strict=True)` | 类似`.crop` 和 `.within_bbox`, 仅保留边界框外部的对象.                                                                                                                                                                                                                                                                                                                                                        |
| `.filter(test_function)`                                   | 过滤对象`.objects`,仅提取当过滤函数`test_function(obj)`返回`True`的对象.                                                                                                                                                                                                                                                                                                                                      |

... 也包含了以下方法:

| 方法       | 描述                                                                                                                                                            |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `.close()` | 默认情况下，"Page"对象会缓存其布局和对象信息，以避免重新处理。不过，在解析大型 PDF 文件时，这些缓存属性可能会占用大量内存。您可以使用此方法刷新缓存并释放内存。 |

更多方法说明请看以下章节:

- [可视化调试](#visual-debugging)
- [提取文本](#extracting-text)
- [提取表格](#extracting-tables)

### 对象

`pdfplumber.PDF` 和 `pdfplumber.Page`实例均提供各种类型的对象, 对象来自 [`pdfminer.six`](https://github.com/pdfminer/pdfminer.six/)PDF 解析。以下属性分别返回匹配对象的 Python 列表：

- `.chars`, 代表单个文本字符.
- `.lines`, 代表一条一维线.
- `.rects`, 代表单个二维矩形.
- `.curves`, 代表一序列`pdfminer.six`无法识别成线或矩形的连接点,.
- `.images`, 代表图像.
- `.annots`, 代表一个 PDF 批注 (详见第 8.4 节 [官方 PDF 规范](https://www.adobe.com/content/dam/acom/en/devnet/acrobat/pdfs/pdf_reference_1-7.pdf))
- `.hyperlinks`, 代表链接注释 `Link`包含`URI`属性

以上每个对象都是字典形式,其包含以下属性:

#### `char` 属性

| 属性                   | 说明                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `page_number`          | 找到此字符的页码。                                                                                                         |
| `text`                 | 文本内容 例: "z", 或 "Z" 或 " ".                                                                                           |
| `fontname`             | 字体                                                                                                                       |
| `size`                 | 字号                                                                                                                       |
| `adv`                  | 等于文本宽度*字体大小*比例因子。                                                                                           |
| `upright`              | 是否垂直                                                                                                                   |
| `height`               | 文本高度                                                                                                                   |
| `width`                | 文本高度                                                                                                                   |
| `x0`                   | 其左侧与页面左侧的距离。                                                                                                   |
| `x1`                   | 其右侧与页面左侧的距离。                                                                                                   |
| `y0`                   | 其下侧与页面底部的距离。                                                                                                   |
| `y1`                   | 其上侧与页面底部的距离。                                                                                                   |
| `top`                  | 其上侧与页面顶部的距离。                                                                                                   |
| `bottom`               | 其下侧与页面顶部的距离。                                                                                                   |
| `doctop`               | 其顶部与文档顶部的距离。                                                                                                   |
| `matrix`               | 其“变换矩阵”。（详见下文）                                                                                                 |
| `mcid`                 | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `tag`                  | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `ncs`                  | TKTK                                                                                                                       |
| `stroking_pattern`     | TKTK                                                                                                                       |
| `non_stroking_pattern` | TKTK                                                                                                                       |
| `stroking_color`       | 字符轮廓颜色 (i.e., stroke). 详情查看 [docs/colors.md](docs/colors.md) .                                                   |
| `non_stroking_color`   | 字符填充颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `object_type`          | 对象类型:"char"                                                                                                            |

**备注**: 文本变换矩阵(`matrix`)说明请参见章节 4.2.2[PDF 参考](https://ghostscript.com/~robin/pdf_reference17.pdf) (第 6 版). 矩阵控制角色的缩放、倾斜和位置平移。旋转是缩放和倾斜的组合，但在大多数情况下可以认为等于 x 轴倾斜。 `pdfplumber.ctm`子模块定义了一个类 `CTM`，该类可以帮助进行这些计算。例如：

```python
from pdfplumber.ctm import CTM
my_char = pdf.pages[0].chars[3]
my_char_ctm = CTM(*my_char["matrix"])
my_char_rotation = my_char_ctm.skew_x
```

#### `line` 属性

| 属性                 | 说明                                                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `page_number`        | 其所在的页码                                                                                                               |
| `height`             | 高度                                                                                                                       |
| `width`              | 宽度                                                                                                                       |
| `x0`                 | 其左侧与页面左侧的距离。                                                                                                   |
| `x1`                 | 其右侧与页面左侧的距离。                                                                                                   |
| `y0`                 | 其下侧与页面底部的距离。                                                                                                   |
| `y1`                 | 其上侧与页面底部的距离。                                                                                                   |
| `top`                | 其上侧与页面顶部的距离。                                                                                                   |
| `bottom`             | 其下侧与页面顶部的距离。                                                                                                   |
| `doctop`             | 其顶部与文档顶部的距离。                                                                                                   |
| `linewidth`          | 线粗                                                                                                                       |
| `stroking_color`     | 线段轮廓颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `non_stroking_color` | 线段填充颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `mcid`               | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `tag`                | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `object_type`        | 对象类型:"line"                                                                                                            |

#### `rect` 属性

| 属性                 | 说明                                                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `page_number`        | 其所在的页码                                                                                                               |
| `height`             | 高度                                                                                                                       |
| `width`              | 宽度                                                                                                                       |
| `x0`                 | 其左侧与页面左侧的距离。                                                                                                   |
| `x1`                 | 其右侧与页面左侧的距离。                                                                                                   |
| `y0`                 | 其下侧与页面底部的距离。                                                                                                   |
| `y1`                 | 其上侧与页面底部的距离。                                                                                                   |
| `top`                | 其上侧与页面顶部的距离。                                                                                                   |
| `bottom`             | 其下侧与页面顶部的距离。                                                                                                   |
| `doctop`             | 其顶部与文档顶部的距离。                                                                                                   |
| `linewidth`          | 线粗                                                                                                                       |
| `stroking_color`     | 矩形边框轮廓颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                              |
| `non_stroking_color` | 矩形填充颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `mcid`               | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `tag`                | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `object_type`        | 对象类型:"rect"                                                                                                            |

#### `curve` 属性

| 属性                 | 说明                                                                                                                       |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `page_number`        | 其所在的页码                                                                                                               |
| `points`             | Points —  描述曲线的点列表包含 `(x, top)` 元组                                                                             |
| `height`             | 曲线边界框的高度。                                                                                                         |
| `width`              | 曲线边界框的宽度。                                                                                                         |
| `x0`                 | 曲线最左侧点与页面左侧的距离。                                                                                             |
| `x1`                 | 曲线最右侧点与页面左侧的距离。                                                                                             |
| `y0`                 | 曲线最低点到页面底部的距离。                                                                                               |
| `y1`                 | 曲线最高点到页面底部的距离。                                                                                               |
| `top`                | 曲线最高点到页面顶部的距离。                                                                                               |
| `bottom`             | 曲线最低点到页面底部的距离。                                                                                               |
| `doctop`             | 曲线最高点到文档顶部的距离。                                                                                               |
| `linewidth`          | 线粗                                                                                                                       |
| `fill`               | 是否填充曲线路径定义的形状.                                                                                                |
| `stroking_color`     | 曲线轮廓颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `non_stroking_color` | 曲线填充颜色. 详情查看 [docs/colors.md](docs/colors.md) .                                                                  |
| `mcid`               | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `tag`                | 如果有此 [标记内容](https://ghostscript.com/~robin/pdf_reference17.pdf#page=850) 则获取段落 ID (否则为 `None`). _实验属性_ |
| `object_type`        | 对象类型:"curve"                                                                                                           |

#### 衍生属性

此外， `pdfplumber.PDF` 和 `pdfplumber.Page`还提供以下两个对象：
`.rect_edges` （将每个矩形分解为四条线）
`.curve_edges` (与`curve`对象相同)
`.edges`（将 `.rect_edges`,`.curve_edges`与`.lines`组合在一起）。

#### `image` 属性

\[待完善.]

### 获取`pdfminer.six`高级布局对象

加载 PDF( `pdfplumber.open(...)`)时通过传参`laparams`打开 PDF 后,每个页面对象将包含 `pdfminer.six`高级布局对象, 例如: `"textboxhorizontal"`.

## 可视化调试

`pdfplumber`的调试工具可以辅助了解 PDF 文档结构,利用工具可以导出对象信息.

### 输出页面图像对象 `PageImage`

使用 `my_page.to_image()`获取页面(包括裁剪页面)的图像对象 `PageImage` .设置分辨率`resolution={integer}`, 默认分辨率:72 例:

使用 `my_page.to_image()`获取页面(包括裁剪页面)的图像对象 `PageImage`. 以下参数可以设置:

- `resolution`: 图片分辨率`resolution={integer}`, 默认分辨率:72. 类型: `int`.
- `width`: 图片宽度(像素).默认:未设置, 取决于`resolution`. 类型: `int`.
- `height`: 图片高度(像素).默认:未设置, 取决于`resolution`. 类型: `int`.
- `resolution`: The desired number pixels per inch. Default: `72`. 类型: `int`.
- `antialias`: 创建图像时是否使用抗锯齿（antialiasing）。设置为`True`可以生成更平滑的文本和图形，但文件大小会变大。默认情况下是`False`. 类型: `bool`.
- `force_mediabox`: 使用页面的 `.mediabox` 尺寸，而不是 `.cropbox` 尺寸。默认：`False`。类型： `bool`。

示例:

```python
im = my_pdf.pages[0].to_image(resolution=150)
```

使用脚本或交互环境时,`im.show()`将使用本地图片浏览器查看图片, `PageImage` 对象也可以很好地与 Jupyter 配合使用；它们会自动渲染为单元格输出。例如：

![使用Jupyter进行可视化调试](examples/screenshots/visual-debugging-in-jupyter.png "Visual debugging in Jupyter")

_备注_: `.to_image(...)` 原型是 `Page.crop(...)`/`CroppedPage` 实例, 但是其无法使用 `Page.filter(...)`/`FilteredPage` 实例进行改变.

### `PageImage` 基础方法

| 方法                                                                           | 说明                                                                                                                                                                                                                                    |
| ------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `im.reset()`                                                                   | 重置,清除已绘制的所有内容。                                                                                                                                                                                                             |
| `im.copy()`                                                                    | 将图像复制到新的“PageImage”对象。                                                                                                                                                                                                       |
| `im.show()`                                                                    | 使用本地图片查看器浏览图片                                                                                                                                                                                                              |
| `im.save(path_or_fileobject, format="PNG")`                                    | 保存已标记的图片                                                                                                                                                                                                                        |
| `im.save(path_or_fileobject, format="PNG", quantize=True, colors=256, bits=8)` | 将带有注释的图像保存为 PNG 文件。默认参数会将图像量化为一个包含 256 种颜色的调色板，并以 8 位色深保存 PNG 文件。如果想禁用量化，可以传递`quantize=False`参数；如果想调整颜色调色板的大小，可以传递`colors=N`参数，其中 N 表示颜色的数量 |

### 绘制方法

你可以传递坐标或`pdfplumber`PDF 对象 (例: char, line, rect)进行绘图.

| 单对象绘制                                                                | 批量绘制                                      | 说明                                                                   |
| ------------------------------------------------------------------------- | --------------------------------------------- | ---------------------------------------------------------------------- |
| `im.draw_line(line, stroke={color}, stroke_width=1)`                      | `im.draw_lines(list_of_lines, **kwargs)`      | 绘制线条根据`line`, `curve`, 或起止点坐标元组(例: `((x, y), (x, y))`). |
| `im.draw_vline(location, stroke={color}, stroke_width=1)`                 | `im.draw_vlines(list_of_locations, **kwargs)` | 在`location`处 x 坐标处绘制一条垂直线                                  |
| `im.draw_hline(location, stroke={color}, stroke_width=1)`                 | `im.draw_hlines(list_of_locations, **kwargs)` | 在`location`处 y 坐标处绘制一条水平线                                  |
| `im.draw_rect(bbox_or_obj, fill={color}, stroke={color}, stroke_width=1)` | `im.draw_rects(list_of_rects, **kwargs)`      | 绘制矩形`rect`, `char`, 等, 或 4 坐标元组边界框.                       |
| `im.draw_circle(center_or_obj, radius=5, fill={color}, stroke={color})`   | `im.draw_circles(list_of_circles, **kwargs)`  | 以`(x, y)`为圆心画圆 `char`, `rect`, 等.                               |

备注:以上绘图方法是基于 Pillow 实现的 [`ImageDraw` methods](http://pillow.readthedocs.io/en/latest/reference/ImageDraw.html), 但是为了与 SVG 的一致性，调整了`fill`/`stroke`/`stroke_width`参数命名.

### 直观调试表格查找器

`im.debug_tablefinder(table_settings={})` 将返回一个页面图像版本，并叠加检测到的线条（红色）、交叉点（圆形）和表格（浅蓝色）。

## 提取文本

`pdfplumber` 可以从页面提取文本 (包括裁剪的或者衍生的页面). 它也尽量尝试保持文本布局, 尽可能的识别单词的坐标及查询条件. `Page`对象可以使用以下方法进行文本提取:

| 方法                                                                                                                                                                                                                           | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `.extract_text(x_tolerance=3, x_tolerance_ratio=None, y_tolerance=3, layout=False, x_density=7.25, y_density=13, **kwargs)`                                                                                                    | 提取页面字符串.<ul><li><p>当 `layout=False`: 如果一个字符的`x1`与下一个字符的`x0`之间的差大于`x_tolerance`，则添加空格。(如果 `x_tolerance_ratio` 不是 `None`，提取器将使用等于 `x_tolerance_ratio * previous_character["size"]` 的动态 `x_tolerance`)。如果一个字符的`doctop`与下一个字符的`doctop`之间的差大于`y_tolerance`，则添加换行符。</p></li><li><p>当 `layout=True` (_实验性功能_): 尝试模拟页面上文本的结构布局，使用`x_density`和`y_density`来确定每个"点"（PDF 的单位）所需的最小字符数/换行符数。所有剩余的`**kwargs`参数会传递给`.extract_words(...)`（见下文），这是计算布局的第一步</p></li></ul>                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `.extract_text_simple(x_tolerance=3, y_tolerance=3)`                                                                                                                                                                           | `.extract_text(...)`的加速版,使用了较为简单的判断逻辑,获取的内容可能没那么精确.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `.extract_words(x_tolerance=3, x_tolerance_ratio=None, y_tolerance=3, keep_blank_chars=False, use_text_flow=False, horizontal_ltr=True, vertical_ttb=True, extra_attrs=[], split_at_punctuation=False, expand_ligatures=True)` | 提取包含所有单词及其边界框的列表。单词被定义为字符序列，其中（对于“直立”字符）一个字符的`x1`和下一个字符的`x0`之间的差小于等于`x_tolerance`，且一个字符的`doctop`和下一个字符的`doctop`之间的差小于等于`y_tolerance`。(如果 `x_tolerance_ratio` 不是 `None`，提取器将使用等于 `x_tolerance_ratio * previous_character["size"]` 的动态 `x_tolerance`)。对于非直立字符，采用类似的方法，但是测量它们之间的垂直距离而不是水平距离。参数`horizontal_ltr`和`vertical_ttb`指示单词是从左到右（对于水平文字）/从上到下（对于垂直文字）阅读。将`keep_blank_chars`设置为`True`将意味着空字符被视为单词的一部分，而不是单词之间的空格。将`use_text_flow`设置为`True`将使用 PDF 的底层字符流作为指导，对单词进行排序和分割，而不是通过 x/y 位置进行预排序（这模拟了在 PDF 中拖动光标突出显示文本的方式；与此相似，顺序不总是看起来很合乎逻辑）。传递一个`extra_attrs`的列表（例如，`["fontname", "size"]`）将限制每个单词中具有完全相同属性值的字符，并且生成的单词字典将指示这些属性。将`split_at_punctuation`设置为`True`将强制在`string.punctuation`中指定的标点符号处分隔标记；或者你可以通过传递一个字符串来指定分隔标点符号的列表，例如<code>`split_at_punctuation='!"&\'()\*+,.:;<=>?@[\]^\\\{\\ | \\}~\'`</code>。除非设置`expand_ligatures=False`，连字（如 ﬁ）将展开为它们的组合字母（例如`fi`）。 |
| `.extract_text_lines(layout=False, strip=True, return_chars=True, **kwargs)`                                                                                                                                                   | _实验性功能_ 返回一个字典列表，代表页面上的文本行。`strip`参数的工作方式类似于 Python 的`str.strip()`方法，它返回去除周围空白的`text`属性。（只在`layout=True`时相关。）将`return_chars`设置为`False`将在返回的文本行字典中排除单个字符对象。剩下的`**kwargs`是你会传递给`.extract_text(layout=True, ...)`的参数。                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `.search(pattern, regex=True, case=True, main_group=0, return_groups=True, return_chars=True, layout=False, **kwargs)`                                                                                                         | \*实验性功能\* 搜索页面中的文本，并返回与查询匹配的所有实例的列表。对于每个实例，响应字典对象包含匹配的文本、任何正则表达式组匹配、边界框坐标和字符对象本身。`pattern`可以是一个已编译的正则表达式、一个未编译的正则表达式或一个非正则字符串。如果`regex`为`False`，则将模式视为非正则字符串。如果`case`为`False`，则以不区分大小写的方式进行搜索。通过设置`main_group`，可以将结果限制在`pattern`中的特定正则表达式组中（默认为 0 表示整个匹配）。将`return_groups`和/或`return_chars`设置为`False`将排除匹配的正则表达式组和/或字符列表（以"groups"和"chars"添加到返回的字典中）。参数`layout`的操作与`.extract_text(...)`相同。剩下的`**kwargs`是你会传递给`.extract_text(layout=True, ...)`的参数。**注意**：零宽度和全空白字符的匹配将被丢弃，因为它们（通常）在页面上没有明确的位置。                                                                                                                                                                                                                                                                                                                                                                                                  |
| `.dedupe_chars(tolerance=1)`                                                                                                                                                                                                   | 删除重复的字符-与其他字符共享相同的文本、字体名称、大小和位置（在“公差”x/y 范围内）-已删除. (参考 [Issue #71](https://github.com/jsvine/pdfplumber/issues/71)了解该机制)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

## 提取表格

`pdfplumber`提取表格大部分借鉴 [Anssi Nurminen 的硕士论文](http://dspace.cc.tut.fi/dpub/bitstream/handle/123456789/21520/Nurminen.pdf?sequence=3), 灵感来源于 [Tabula](https://github.com/tabulapdf/tabula-extractor/issues/16). 工作流程如下:

1.  针对 PDF 页面, 查找（a）明确定义的行和/或（b）页面上单词对齐所隐含的行。

2.  合并重叠或接近重叠的线条.

3.  找出这些线条的交点.

4.  找到使用这些交点作为顶点的最细粒度的矩形集（即单元格）。

5.  将连续单元格分组到表中。

### 表格提取方法

`pdfplumber.Page`页面对象支持以下方法:

| 方法                                    | 说明                                                                                                                                          |
| --------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `.find_tables(table_settings={})`       | 返回表格 `Table` 对象列表. `Table`包含`.cells`, `.rows`,`.bbox` 属性, 以及方法`.extract(x_tolerance=3, y_tolerance=3)`                        |
| `.find_table(table_settings={})`        | 类似`.find_tables(...)`, 只返回页面中*最大*的表格(`Table`表格对象). 如果多个表格尺寸一致 —  相同的单元格数量 —  该方法返回最靠页面顶部的表格. |
| `.extract_tables(table_settings={})`    | 返回从页面上找到的*所有*表中提取的文本，表示为列表列表列表，其结构为“表->行->单元格”。                                                        |
| `.extract_table(table_settings={})`     | 返回从页面上*最大*表(参考`.find_tables(...)`)中提取的文本，该表表示为列表列表，其结构为“行->单元格”。                                         |
| `.debug_tablefinder(table_settings={})` | 返回`TableFinder`类的实例，该类可以访问`.edges`(边), `.intersections`(交点), `.cells`(单元格)和`.tables`(表格)属性.                           |

举例:

```python
pdf = pdfplumber.open("path/to/my.pdf")
page = pdf.pages[0]
page.extract_table()
```

[查看更多示例](examples/notebooks/extract-table-ca-warn-report.ipynb)

### 表格提取配置

默认情况下， `extract_tables`使用页面的垂直线和水平线（或矩形边）作为单元格分隔符。但该方法可以通过 `table_settings`参数进行高度自定义。以下是设置参数及默认值：

```python
{
    "vertical_strategy": "lines",
    "horizontal_strategy": "lines",
    "explicit_vertical_lines": [],
    "explicit_horizontal_lines": [],
    "snap_tolerance": 3,
    "snap_x_tolerance": 3,
    "snap_y_tolerance": 3,
    "join_tolerance": 3,
    "join_x_tolerance": 3,
    "join_y_tolerance": 3,
    "edge_min_length": 3,
    "min_words_vertical": 3,
    "min_words_horizontal": 1,
    "intersection_tolerance": 3,
    "intersection_x_tolerance": 3,
    "intersection_y_tolerance": 3,
    "text_tolerance": 3,
    "text_x_tolerance": 3,
    "text_y_tolerance": 3,
    "text_*": …, # 见下文
}
```

| 配置项                                                                                 | 说明                                                                                                                                                  |
| -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"vertical_strategy"`                                                                  | 垂直策略,可选值 `"lines"`, `"lines_strict"`, `"text"`, `"explicit"`.见后续说明                                                                        |
| `"horizontal_strategy"`                                                                | 水平策略,可选值 `"lines"`, `"lines_strict"`, `"text"`, `"explicit"`. 见后续说明                                                                       |
| `"explicit_vertical_lines"`                                                            | 明确划分表中单元格的垂直线列表。可与上述任何策略结合使用。列表中的项目应该是数字（表示一条直线的`x`坐标，即页面的全高）或 `line`/`rect`/`curve`对象。 |
| `"explicit_horizontal_lines"`                                                          | 明确划分表中单元格的水平线列表。可与上述任何策略结合使用。列表中的项目应该是数字（表示一条直线的`y`坐标即页面的全高）或 `line`/`rect`/`curve`对象。   |
| `"snap_tolerance"`, `"snap_x_tolerance"`, `"snap_y_tolerance"`                         | `snap_tolerance`像素内的平行线将“捕捉”到相同的水平或垂直位置。                                                                                        |
| `"join_tolerance"`, `"join_x_tolerance"`, `"join_y_tolerance"`                         | 同一条无限线上的线段，其端点在彼此的`join_tolerance`范围内，将“连接”为一条线段                                                                        |
| `"edge_min_length"`                                                                    | 在尝试重建表之前，将丢弃小于`edge_min_length`的边                                                                                                     |
| `"min_words_vertical"`                                                                 | 使用 `"horizontal_strategy": "text"`时，至少 `min_words_horizontal`单词必须共享相同的对齐方式。                                                       |
| `"min_words_horizontal"`                                                               | 使用 `"horizontal_strategy": "text"`时,至少`min_words_horizontal`单词必须共享相同的对齐方式。                                                         |
| `"intersection_tolerance"`, `"intersection_x_tolerance"`, `"intersection_y_tolerance"` | 将边组合到单元格中时，正交边必须在`intersection_tolerance` 像素范围内才能视为相交。                                                                   |
| `"text_*"`                                                                             | 以`text_`为前缀的配置项将在表格文本提取时生效. 可以设置`Page.extract_text(...)`中的参数.                                                              |
| `"text_x_tolerance"`, `"text_y_tolerance"`                                             | `text_`为前缀的配置项也适用于表格识别算法,也就是说，当该算法搜索单词时，它将期望每个单词中的单个字母之间的距离不超过`text\_[x                         | y]\_tlerance`像素. |

### 表格提取策略

`vertical_strategy`(垂直策略) 及 `horizontal_strategy`(水平策略)均包含以下选项:

| 策略             | 说明                                                                                                                                                                             |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"lines"`        | 使用页面的图形线（包括矩形对象的边）作为潜在表格单元格的边框。                                                                                                                   |
| `"lines_strict"` | 使用页面的图形线（而不是矩形对象的边）作为潜在表格单元格的边框。                                                                                                                 |
| `"text"`         | 对于`vertical_strategy`（垂直策略）：推导连接页面上单词的左、右或中心的（假想的）行，并将这些行用作潜在表格单元格的边框。对于`horizontal_strategy`，相同操作，但使用单词的顶部。 |
| `"explicit"`     | 仅使用`explicit_vertical_lines` / `explicit_horizontal_lines`中明确定义的线。                                                                                                    |

### 备注

- 在尝试提取表之前裁剪页面通常很有帮助 — `Page.crop(bounding_box)`

- `v0.5.0`版本的`pdfplumber`针对表格提取功能进行了彻底的重新设计，并引入了突破性的更改。

## 提取表单值

有时 PDF 文件可以包含表单，包含了人们可以填写和保存的输入。虽然表单字段中的值与 PDF 文件中的其他文本类似，但表单数据的处理方式不同。如果您想彻底了解详细信息，请参阅本手册第 671 页 [specification](https://opensource.adobe.com/dc-acrobat-sdk-docs/pdfstandards/pdfreference1.7old.pdf).

`pdfplumber`没有用于处理表单数据的接口，但您可以使用`pdfplumber`对`pdfminer`的封装组件来访问。

例如，此代码段将检索表单字段名和值，并将它们存储在字典中。

```python
import pdfplumber
from pdfplumber.utils.pdfinternals import resolve_and_decode, resolve
pdf = pdfplumber.open("document_with_form.pdf")

def parse_field_helper(form_data, field, prefix=None):
    """ appends any PDF AcroForm field/value pairs in `field` to provided `form_data` list

        if `field` has child fields, those will be parsed recursively.
    """
    resolved_field = field.resolve()
    field_name = '.'.join(filter(lambda x: x, [prefix, resolve_and_decode(resolved_field.get("T"))]))
    if "Kids" in resolved_field:
        for kid_field in resolved_field["Kids"]:
            parse_field_helper(form_data, kid_field, prefix=field_name)
    if "T" in resolved_field or "TU" in resolved_field:
        # "T" is a field-name, but it's sometimes absent.
        # "TU" is the "alternate field name" and is often more human-readable
        # your PDF may have one, the other, or both.
        alternate_field_name  = resolve_and_decode(resolved_field.get("TU")) if resolved_field.get("TU") else None
        field_value = resolve_and_decode(resolved_field["V"]) if 'V' in resolved_field else None
        form_data.append([field_name, alternate_field_name, field_value])


form_data = []
fields = resolve(pdf.doc.catalog["AcroForm"])["Fields"]

for field in fields:
    form_data[field_name] = field_value
```

运行以上代码, 得到的`form_data`是一个包含三元素元组的列表. 例如, 一个包含城市和州值域的 PDF 提取出来的表单如下.

```
[
 ['STATE.0', 'enter STATE', 'CA'],
 ['section 2  accident infoRmation.1.0',
  'enter city of accident',
  'SAN FRANCISCO']
]
```

## 用例演示

- [提取加利福尼亚工人调整和再培训通知（WARN）报告中的表格](examples/notebooks/extract-table-ca-warn-report.ipynb). 演示基本的可视化调试和表提取。

- [提取 FBI 的国家即时刑事背景调查系统 PDF 文件中的表格](examples/notebooks/extract-table-nics.ipynb). 演示如何使用可视化调试查找最佳的表提取设置。还演示了 `Page.crop(...)`(页面裁剪) 及 `Page.extract_text(...).`(文本导出)

- [检查和可视化“curve”曲线对象](examples/notebooks/ag-energy-roundup-curves.ipynb).

- [从圣何塞警察局枪支搜索报告中提取等宽数据](examples/notebooks/san-jose-pd-firearm-report.ipynb), `Page.extract_text(...)`示例.

## 与其他库的比较

其他几个 Python 库帮助用户从 PDF 中提取信息。综上所述，`pdfplumber`通过结合以下功能而区别于其他 PDF 处理库：

- 可以轻松访问每个 PDF 对象的详细信息

- 拥有用于提取文本和表格的更高级、可自定义的方法

- 集成了可视化调试

- 其他有用的实用功能，例如通过裁剪框过滤对象

了解`pdfplumber`无法提供的功能也很有帮助：

- 创建 PDF

- 修改 PDF

- 文字识别(OCR)

- 完全支持从 OCR 识别过的 PDF 提取表格

### 特性比较

- [`pdfminer.six`](https://github.com/pdfminer/pdfminer.six) `pdfplumber`是基于该库实现的。它主要关注解析 PDF、分析 PDF 布局和对象定位，以及提取文本。但它不提供用于表提取或可视化调试的工具。

- [`PyPDF2`](https://github.com/mstamy2/PyPDF2) 是一个纯 Python 库，“能够拆分、合并、裁剪和转换 PDF 文件的页面。它还可以向 PDF 文件添加自定义数据、查看选项和密码。”它可以提取页面文本，但无法轻松访问形状对象（矩形、线条等）、表提取或可视化调试工具。

- [`pymupdf`](https://pymupdf.readthedocs.io/) 运行速度比`pdfminer`快。可以生成和修改 PDF，但该库需要安装非 Python 软件（MuPDF）。它无法轻松访问形状对象（矩形、直线等），并且不提供表提取或可视化调试工具。

- [`camelot`](https://github.com/camelot-dev/camelot), [`tabula-py`](https://github.com/chezou/tabula-py), and [`pdftables`](https://github.com/drj11/pdftables) 这些库主要关注表格提取。在某些情况下，它们可能更适合提取特定的表格。

## 认可 / 贡献

非常感谢以下提供了想法、功能和修复的用户：

- [Jacob Fenton](https://github.com/jsfenfen)
- [Dan Nguyen](https://github.com/dannguyen)
- [Jeff Barrera](https://github.com/jeffbarrera)
- [Bob Lannon](https://github.com/boblannon)
- [Dustin Tindall](https://github.com/dustindall)
- [@yevgnen](https://github.com/Yevgnen)
- [@meldonization](https://github.com/meldonization)
- [Oisín Moran](https://github.com/OisinMoran)
- [Samkit Jain](https://github.com/samkit-jain)
- [Francisco Aranda](https://github.com/frascuchon)
- [Kwok-kuen Cheung](https://github.com/cheungpat)
- [Marco](https://github.com/ubmarco)
- [Idan David](https://github.com/idan-david)
- [@xv44586](https://github.com/xv44586)
- [Alexander Regueiro](https://github.com/alexreg)
- [Daniel Peña](https://github.com/trifling)
- [@bobluda](https://github.com/bobluda)
- [@ramcdona](https://github.com/ramcdona)
- [@johnhuge](https://github.com/johnhuge)
- [Jhonatan Lopes](https://github.com/jhonatan-lopes)
- [Ethan Corey](https://github.com/ethanscorey)
- [Shannon Shen](https://github.com/lolipopshock)
- [Matsumoto Toshi](https://github.com/toshi1127)
- [John West](https://github.com/jwestwsj)
- [David Huggins-Daines](https://github.com/dhdaines)
- [Jeremy B. Merrill](https://github.com/jeremybmerrill)
- [Echedey Luis](https://github.com/echedey-ls)
- [Andy Friedman](https://github.com/afriedman412)

## 贡献者

欢迎贡献代码，由于该库正在积极维护, 请先提交提案

主要维护者:

- [Jeremy Singer-Vine](https://github.com/jsvine)

- [Samkit Jain](https://github.com/samkit-jain)
