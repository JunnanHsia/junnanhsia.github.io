---
title: SpringBoot Word模板导出功能&多行复制功能的实现
description: Word模板导出框架:deepoove公司的poi-tl框架使用.以及自实现的拓展功能:Word中的表格需要进行多行的数据复制时的实现类;
date: 2025-04-10 10:00:39 +0800
categories: [后端, 高级操作] # 文章分类
tags: [SpringBoot,导出,poi] # 文章标签
toc: true # 是否开启右侧的标题导航
comments: true # 是否开启评论
mermaid: true # 是否支持文字生成图表的功能
math: true # 是否支持数学工时
pin: false # 需要指定的后为true
image:
  path: /assets/img/posts/SpringBoot_POI_cartoon.webp # 主图路径宽高为:1200 x 630 或者比例为: 1.91 : 1
  alt: SpringBoot Word模板导出功能&多行复制功能的实现
---

## 一.基础使用
### 0.简介
> **详细介绍见[官方文档](https://deepoove.com/poi-tl)**,poi-tl（poi template language）是Word模板引擎，使用模板和数据创建很棒的Word文档。

#### 现有Word模板导出方案对比:

|    **方案**    |          **移植性**          | **功能性**                                              | **易用性**                                                                                             |
| :------------: | :--------------------------: | :------------------------------------------------------ | :----------------------------------------------------------------------------------------------------- |
|     Poi-tl     |          Java跨平台          | Word模板引擎，基于Apache POI，提供更友好的API           | 低代码，准备文档模板和数据即可                                                                         |
|   Apache POI   |          Java跨平台          | Apache项目，封装了常见的文档操作，也可以操作底层XML结构 | 文档不全，这里有一个教程：[Apache POI Word快速入门](https://deepoove.com/poi-tl/apache-poi-guide.html) |
|   Freemarker   |          XML跨平台           | 仅支持文本，很大的局限性                                | 不推荐，XML结构的代码几乎无法维护                                                                      |
|   OpenOffice   |  部署OpenOffice，移植性较差  | -                                                       | 需要了解OpenOffice的API                                                                                |
| HTML浏览器导出 | 依赖浏览器的实现，移植性较差 | HTML不能很好的兼容Word的格式，样式糟糕                  | -                                                                                                      |
| Jacob、winlib  |         Windows平台          | -                                                       | 复杂，完全不推荐使用                                                                                   |

#### poi-tl具体的功能

| **Word模板引擎功能** | **描述   **                                                                                                              |
| :------------------: | :----------------------------------------------------------------------------------------------------------------------- |
|         文本         | 将标签渲染为文本                                                                                                         |
|         图片         | 将标签渲染为图片                                                                                                         |
|         表格         | 将标签渲染为表格                                                                                                         |
|         列表         | 将标签渲染为列表                                                                                                         |
|         图表         | 条形图（3D条形图）、柱形图（3D柱形图）、面积图（3D面积图）、折线图（3D折线图）、雷达图、饼图（3D饼图）、散点图等图表渲染 |
|   If Condition判断   | 根据条件隐藏或者显示某些文档内容（包括文本、段落、图片、表格、列表、图表等）                                             |
|   Foreach Loop循环   | 根据集合循环某些文档内容（包括文本、段落、图片、表格、列表、图表等）                                                     |
|      Loop表格行      | 循环复制渲染表格的某一行                                                                                                 |
|      Loop表格列      | 循环复制渲染表格的某一列                                                                                                 |
|     Loop有序列表     | 支持有序列表的循环，同时支持多级列表                                                                                     |
|  Highlight代码高亮   | word中代码块高亮展示，支持26种语言和上百种着色样式                                                                       |
|       Markdown       | 将Markdown渲染为word文档                                                                                                 |
|       Word批注       | 完整的批注功能，创建批注、修改批注等                                                                                     |
|       Word附件       | Word中插入附件                                                                                                           |
|     SDT内容控件      | 内容控件内标签支持                                                                                                       |
|    Textbox文本框     | 文本框内标签支持                                                                                                         |
|       图片替换       | 将原有图片替换成另一张图片                                                                                               |
|  书签、锚点、超链接  | 支持设置书签，文档内锚点和超链接功能                                                                                     |
| Expression Language  | 完全支持SpringEL表达式，可以扩展更多的表达式：OGNL, MVEL…​                                                               |
|         样式         | 模板即样式，同时代码也可以设置样式                                                                                       |
|       模板嵌套       | 模板包含子模板，子模板再包含子模板                                                                                       |
|         合并         | Word合并Merge，也可以在指定位置进行合并                                                                                  |
| 用户自定义函数(插件) | 插件化设计，在文档任何位置执行函数                                                                                       |


### 1.框架依赖引入
```xml
<dependency>
  <groupId>com.deepoove</groupId>
  <artifactId>poi-tl</artifactId>
  <version>1.12.2</version>
</dependency>
```

### 2.基础使用
#### 2.2.1 创建word文件模板,加入数据标记
[模板文件](https://junnanhsia.github.io/assets/files/word导出.docx)

![文件内容示例](/assets/img/posts/word_export_template.png)
#### 2.2.2 代码数据填充
```java
    @SneakyThrows
    @PostMapping("export")
    public void export() {
        InputStream templateIns = ResourceUtil.getStream("templates/word导出.docx");//模板文件放置在项目中的resources目录下
        Map<String, Object> exportMap = new HashMap<>();
        /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓  导出数据填充 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
        exportMap.put("title", "测试标题");
        exportMap.put("content", "测试内容");
        //图片填充
        exportMap.put("img1", Pictures.ofUrl("http://www.baidu.com/img/bdlogo.png").size(20, 20).create());
        exportMap.put("img2", Pictures.ofUrl("http://rongcloud-web.qiniudn.com/docs_demo_rongcloud_logo.png").size(20, 20).create());
        //循环行数据填充
        exportMap.put("students",new ArrayList<Map<String, Object>>(){{
            add(new HashMap<String, Object>(){{
                put("name", "张三");
                put("gender", "男");
                put("age", "20");
                put("header", Pictures.ofUrl("http://www.baidu.com/img/bdlogo.png").size(20, 20).create());
            }});
            add(new HashMap<String, Object>(){{
                put("name", "李四");
                put("gender", "男");
                put("age", "21");
                put("header", Pictures.ofUrl("http://rongcloud-web.qiniudn.com/docs_demo_rongcloud_logo.png").size(20, 20).create());
            }});
        }});
        //区块对数据填充
        exportMap.put("person",new ArrayList<Map<String, Object>>(){{
            add(new HashMap<String, Object>(){{
                put("name", "张三");
                put("gender", "男");
                put("age", "20");
                put("header", Pictures.ofUrl("http://www.baidu.com/img/bdlogo.png").size(20, 20).create());
            }});
            add(new HashMap<String, Object>(){{
                put("name", "李四");
                put("gender", "男");
                put("age", "21");
                put("header", Pictures.ofUrl("http://rongcloud-web.qiniudn.com/docs_demo_rongcloud_logo.png").size(20, 20).create());
            }});
        }});
        /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑  导出数据填充 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */
        //创建行循环策略
        LoopRowTableRenderPolicy rowTableRenderPolicy = new LoopRowTableRenderPolicy();
        Configure configure = Configure.builder()
                .bind("students", rowTableRenderPolicy) //循环行数据绑定
                //区块对是默认插件,不需要主动申明
                .build();
        XWPFTemplate template = XWPFTemplate.compile(templateIns, configure)
                .render(exportMap);
        String fileName = DateUtil.format(new Date(), "yyyyMMddHHmmss") + "-导出.docx";
        setFileName(response, fileName);
        //写回到响应流中
        OutputStream out = response.getOutputStream();
        BufferedOutputStream bos = new BufferedOutputStream(out);
        template.write(bos);
        bos.flush();
        out.flush();
        PoitlIOUtils.closeQuietlyMulti(template, bos, out);
    }

    //设置下载文件名
    @SneakyThrows
    private void setFileName(HttpServletResponse response, String fileName) {
        StringBuilder contentDispositionValue = new StringBuilder();
        contentDispositionValue.append("attachment; filename=").append(URLUtil.encode(fileName, "UTF-8")).append(";").append("filename*=").append("utf-8''").append(URLUtil.encode(fileName, "UTF-8"));
        response.addHeader("Access-Control-Allow-Origin", "*");
        response.addHeader("Access-Control-Expose-Headers", "Content-Disposition,download-filename");
        response.setHeader("Content-disposition", contentDispositionValue.toString());
        response.setHeader("download-filename", URLUtil.encode(fileName, "UTF-8"));
        response.setContentType("application/octet-stream");
    }
```

#### 3.填充结果
![填充结果](/assets/img/posts/word_export_result.png)

更多的填充插件功能,参考官网的示例.

## 二.拓展Word中表格多行复制
官方的拓展插件中虽然可以使用区块对,进行多行的循环,但是区块对针对表格中的部分内容(例如某两行)进行循环,则不支持. 所以此块需要使用自实现的多行表格的循环处理;
![表格多行循环模板定义](/assets/img/posts/word_export_mutirow_template.png)
### 模板内容
![模板内容](/assets/img/posts/word_export_template1.png)

### 插件
```java
/*循环复制多行的策略*/
public static class LoopMultiRowTableRenderPolicy implements RenderPolicy {
    private String itemPrefix;
    private String itemSuffix;

    private int startRowStart;//起始行
    private int startRowEnd;//结束行
    private int rowDiff;//行数差异


    public LoopMultiRowTableRenderPolicy() {
        this("[", "]");
    }

    public LoopMultiRowTableRenderPolicy(String itemPrefix, String itemSuffix) {
        this.itemPrefix = itemPrefix;
        this.itemSuffix = itemSuffix;
    }
    
    /**
     * @description 深入拷贝指定行, 返回新的行数据
     * @create: 2025/3/13 11:05
     **/
    @SneakyThrows
    private XWPFTableRow deepCloneRow(XWPFTable table, XWPFTableRow row) {
        CTRow ctrow = CTRow.Factory.parse(row.getCtRow().newInputStream());//重点行
        XWPFTableRow newRow = new XWPFTableRow(ctrow, table);
        return newRow;
    }

    /**
     * @description 根据行的模板标记, 填充数据
     * @create: 2025/4/10 17:13
     **/
    @SneakyThrows
    private void fillRowData(XWPFTableRow row, Map<String, Object> rowData) {
        List<XWPFTableCell> tableCells = row.getTableCells();// 在新增的行上面创建cell
        for (XWPFTableCell cell : tableCells) {
            for (XWPFParagraph paragraph : cell.getParagraphs()) {
                List<XWPFRun> runs = paragraph.getRuns();
                for (int i = 0; i < runs.size(); i++) {
                    XWPFRun r = runs.get(i);
                    String text = r.getText(0);
                    if (StrUtil.isNotEmpty(text)) {
                        String pureText = text.replace(itemPrefix, "")
                                .replace(itemSuffix, "")
                                .replace("@", "");
                        if (rowData.containsKey(pureText)) {
                            Object rowDataItem = rowData.get(pureText);
                            if (rowDataItem instanceof String) {
                                r.setText((String) rowDataItem, 0);//要深入到原cell中的run替换内容才能保证样式一致
                            }
                            if (rowDataItem instanceof PictureRenderData) {
                                XWPFRun run = paragraph.createRun();
                                PictureRenderData picture = (PictureRenderData) rowDataItem;
                                PictureRenderPolicy.Helper.renderPicture(run, picture);
                                paragraph.removeRun(i);//移除可能已经存在的图片
                            }
                        }
                    }
                }
            }
        }
    }

    @Override
    public void render(ElementTemplate eleTemplate, Object data, XWPFTemplate template) {
        RunTemplate runTemplate = (RunTemplate) eleTemplate;
        XWPFRun run = runTemplate.getRun();
        try {
            if (!TableTools.isInsideTable(run)) {
                throw new IllegalStateException(
                        "The template tag " + runTemplate.getSource() + " must be inside a table");
            }
            XWPFTableCell startTagCell = (XWPFTableCell) ((XWPFParagraph) run.getParent()).getBody();
            XWPFTable table = startTagCell.getTableRow().getTable();
            run.setText("", 0);

            startRowStart = getTemplateRowIndex(startTagCell);//开始行的索引
            // 找到结束行的索引
            int templateRowIndexStartEnd = getTemplateRowIndexByTemplateTag(table, startRowStart, "↑" + runTemplate.getTagName(), true);
            if (templateRowIndexStartEnd == -1) {
                throw new IllegalStateException(
                        "The template end tag " + runTemplate.getSource() + " not found in table");
            }

            /* ↓↓↓↓↓↓ 模板行数据填充 ↓↓↓↓↓↓ */
            rowDiff = templateRowIndexStartEnd - startRowStart + 1;
            Map<Integer, XWPFTableRow> templateRowsMap = new HashMap<>();
            for (int i = 0; i < rowDiff; i++) {
                templateRowsMap.put(i, table.getRow(startRowStart + i));
            }
            /* ↑↑↑↑↑↑ 模板行数据填充 ↑↑↑↑↑↑ */
            List<Map<String, Object>> dataMaps = (List<Map<String, Object>>) data;
            for (int i = 0; i < dataMaps.size(); i++) {
                Map<String, Object> rowData = dataMaps.get(i);
                //需要复制的具体行
                for (int j = 0; j < rowDiff; j++) {
                    XWPFTableRow templateRow = templateRowsMap.get(j);
                    //复制模板行
                    if (i != dataMaps.size() - 1) {
                        XWPFTableRow copyRow = deepCloneRow(table, templateRow);
                        fillRowData(copyRow, dataMaps.get(i + 1));
                        table.addRow(copyRow, startRowStart + ((i + 1) * rowDiff) + j);
                    }
                    //填充数据
                    if (i == 0) {
                        // 填充templateRow
                        fillRowData(templateRow, rowData);
                    }
                }
            }
        } catch (Exception e) {
            throw new RenderException("HackLoopTable for " + eleTemplate + "error: " + e.getMessage(), e);
        }
    }

    private int getTemplateRowIndex(XWPFTableCell tagCell) {
        XWPFTableRow tagRow = tagCell.getTableRow();
        return getRowIndex(tagRow);
    }

    private int getTemplateRowIndexByTemplateTag(XWPFTable table, int templateRowIndexStart, String templateTag, boolean clearTag) {
        int index = -1;
        List<XWPFTableRow> rows = table.getRows();
        for (int i = templateRowIndexStart; i < rows.size(); i++) {
            XWPFTableRow xwpfTableRow = rows.get(i);
            List<ICell> tableICells = xwpfTableRow.getTableICells();
            for (ICell tableICell : tableICells) {
                if (tableICell instanceof XWPFTableCell) {
                    String text = ((XWPFTableCell) tableICell).getText();
                    if (text.contains(templateTag)) {
                        //清空模板标记的内容
                        if (clearTag) {
                            String nText = text.replace("{{" + templateTag + "}}", "");
                            List<XWPFParagraph> paragraphs = ((XWPFTableCell) tableICell).getParagraphs();
                            for (XWPFParagraph paragraph : paragraphs) {
                                List<XWPFRun> runs = paragraph.getRuns();
                                for (XWPFRun run : runs) {
                                    run.setText(nText, 0);//注意在模板上,在对应的位置上使用任务文本占位,此处才能在这个pos然后进行内容的替换
                                }
                            }
                        }
                        return i;
                    }
                }
            }
        }
        return index;
    }

    private int getRowIndex(XWPFTableRow row) {
        List<XWPFTableRow> rows = row.getTable().getRows();
        return rows.indexOf(row);
    }

}
```

### 插件使用
```java
Map<String, Object> exportMap = new HashMap<>();
/* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓  导出数据填充 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
//自定义多行循环
exportMap.put("studentsSelf", new ArrayList<Map<String, Object>>() {{
    add(new HashMap<String, Object>() {{
        put("name", "张三");
        put("gender", "男");
        put("age", "20");
        put("header", Pictures.ofUrl("http://www.baidu.com/img/bdlogo.png").size(20, 20).create());
    }});
    add(new HashMap<String, Object>() {{
        put("name", "李四");
        put("gender", "男");
        put("age", "21");
        put("header", Pictures.ofUrl("http://rongcloud-web.qiniudn.com/docs_demo_rongcloud_logo.png").size(20, 20).create());
    }});
    add(new HashMap<String, Object>() {{
        put("name", "王五");
        put("gender", "女");
        put("age", "22");
        put("header", Pictures.ofUrl("http://www.baidu.com/img/bdlogo.png").size(20, 20).create());
    }});
}});
/* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑  导出数据填充 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */
LoopMultiRowTableRenderPolicy multirowTableRenderPolicy = new LoopMultiRowTableRenderPolicy();
Configure configure = Configure.builder()
        .addPlugin('↓', rowTableRenderPolicy) //循环行数据绑定
        .bind("studentsSelf", multirowTableRenderPolicy) //自定义多行数据复制插件
        //区块对是默认插件,不需要主动申明
        .build();
XWPFTemplate template = XWPFTemplate.compile(templateIns, configure)
        .render(exportMap);
String fileName = DateUtil.format(new Date(), "yyyyMMddHHmmss") + "-导出.docx";
setFileName(response, fileName);//基础使用中的公共方法
//写回到响应流中
OutputStream out = response.getOutputStream();
BufferedOutputStream bos = new BufferedOutputStream(out);
template.write(bos);
bos.flush();
out.flush();
PoitlIOUtils.closeQuietlyMulti(template, bos, out);
```

### 填充结果
![填充结果](/assets/img/posts/word_export_result1.png)