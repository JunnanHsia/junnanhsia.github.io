---
title: SpringBoot_Word转PDF功能
description: 使用aspose-words和poi-tl框架完成,实现了使用在线的导出模板自动的填充模板数据,而后导出word或者pdf文件;
date: 2025-12-25 22:22:13 +0800
categories: [后端, 高级操作] # 文章分类
tags: [SpringBoot,导出,poi] # 文章标签
toc: true # 是否开启右侧的标题导航
comments: true # 是否开启评论
mermaid: true # 是否支持文字生成图表的功能
math: true # 是否支持数学公式
pin: false # 需要指定的后为true
image:
  path: /assets/posts/2025-12-25-SpringBoot_Word转PDF功能/poster.webp # 主图路径宽高为:1200 x 630 或者比例为: 1.91 : 1
  alt: SpringBoot_Word转PDF功能
---

## 一.基础使用
### 0.使用流程
- 编辑word模板,将模板上传到oss服务中,取得模板文件地址, word模板标记详见[poi-tl官方文档](https://deepoove.com/poi-tl)
- 项目中集成依赖,引入工具类
- 使用工具类操作,导出填充后的word文件,或者pdf文件

### 1.框架依赖引入
#### 1.1依赖内容
```xml
<!--word模板填充-->
<dependency>
  <groupId>com.deepoove</groupId>
  <artifactId>poi-tl</artifactId>
  <version>1.12.2</version>
</dependency>
<!--   word转pdf,jar包放置到resources/lib目录下   -->
<dependency>
    <groupId>aspose</groupId>
    <artifactId>aspose-words</artifactId>
    <version>20.12</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/lib/aspose-words-20.12-jdk17-cracked.jar</systemPath>
    <!-- 原版最新 -->
    <!-- <systemPath>${project.basedir}/lib/aspose-words-24.6-jdk17.jar</systemPath> -->
</dependency>
<!--依赖中可能存在版本冲突问题,使用IDEA插件Maven Helper,排查解决-->
```
#### 1.2 aspose-words框架jar包以及授权的文件
[JAR包文件](/assets/posts/2025-12-25-SpringBoot_Word转PDF功能/aspose-words-20.12-jdk17-cracked.jar) 文件放置到项目的resources/lib目录下

[JAR包文件授权文件](/assets/posts/2025-12-25-SpringBoot_Word转PDF功能/license.xml)文件放置到resources目录下

### 2.基础使用
#### 2.1 导出工具类
```java
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.DateUtil;
import cn.hutool.core.io.FileUtil;
import cn.hutool.core.map.MapUtil;
import cn.hutool.core.util.ReflectUtil;
import cn.hutool.http.HttpUtil;
import com.alibaba.fastjson.JSON;
import com.aspose.words.Shape;
import com.aspose.words.*;
import com.deepoove.poi.XWPFTemplate;
import com.deepoove.poi.config.Configure;
import com.deepoove.poi.config.ConfigureBuilder;
import com.deepoove.poi.plugin.table.LoopRowTableRenderPolicy;
import com.deepoove.poi.policy.RenderPolicy;
import com.deepoove.poi.template.MetaTemplate;
import com.deepoove.poi.template.run.RunTemplate;
import com.deepoove.poi.util.PoitlIOUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.core.io.ClassPathResource;
import org.springframework.core.io.Resource;

import java.awt.*;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.*;
import java.util.List;
import java.util.stream.Collectors;

/**
 * @description word, pdf操作工具类
 **/
@Slf4j
public class WordPdfUtil {

  public static void main(String[] args) {
    //测试
    //模板1:
    String templateName = "1.机械设备买卖合同范本修订（修订）.docx";
    String templateUrl = "https://stage-minio1-view-dic.diccp.com/ctcemti-ggc-default/word_template_jx_1.docx";
    HashMap<String, Object> params = new HashMap<String, Object>() {{
      put("number", "1123");
      put("name", "测试工程");
      put("buy", "张三");
      put("sale", "天恩科技");
      put("location", "北京市海淀区");
      put("year", "2025");
      put("month", "9");
      put("day", "9");
      put("LoopRow_Devices", new ArrayList<Map<String, Object>>() {{
        add(new HashMap<String, Object>() {{
          put("name", "设备1");
          put("xh", "型号xx");
          put("unit", "个");
          put("num", "1");
          put("noTaxPrice", "11");
          put("hasTaxPrice", "22");
          put("noTaxAll", "33");
          put("factory", "天恩");
          put("remark", "无备注");
        }});
        add(new HashMap<String, Object>() {{
          put("name", "设备2");
          put("xh", "型号2xx");
          put("unit", "个");
          put("num", "2");
          put("noTaxPrice", "22");
          put("hasTaxPrice", "33");
          put("noTaxAll", "55");
          put("factory", "天恩2");
          put("remark", "无备注1");
        }});
      }});
      put("zzTax", "123");
      put("hasTaxAll", "456");
      //小计类
      put("xhXj", "11");
      put("unitXj", "22");
      put("numXj", "33");
      put("noTaxPriceXj", "44");
      put("hasTaxPriceXj", "55");
      put("noTaxAllXj", "66");
    }};
    String fileUrl = inflateWordTemplate(templateUrl, params, WORD_TRANS_TYPE.WORD);
    log.info("模板:{} 文件路径:{}", templateName, fileUrl);
    fileUrl = inflateWordTemplate(templateUrl, params, WORD_TRANS_TYPE.PDF);
    log.info("模板:{} 文件路径:{}", templateName, fileUrl);


  }

  public enum WORD_TRANS_TYPE {
    WORD, PDF
  }

  /**
   * @description 特殊的渲染策略的key前缀和对应的渲染策略类, TODO 如果有特殊的格式类型,需要在此处进行追加;
   **/
  public static final Map<String, Class<? extends RenderPolicy>> SPECIAL_KEY_MAP_CLASS = new HashMap<String, Class<? extends RenderPolicy>>() {{
    put("LoopRow", LoopRowTableRenderPolicy.class);
  }};

  /**
   * @description: 填充word模板文件参数
   * @author: Jeff@夏俊男
   * @date: 2025/9/8 09:58
   * @Param templateUrl: 模板文件地址
   * @Param params: 模板文件参数
   * @Param type: 输出文件类型枚举,WORD,PDF
   * @return: java.lang.String 填充了模板参数的文件路径
   **/
  public static String inflateWordTemplate(String templateUrl, Map<String, Object> params, WORD_TRANS_TYPE type) {
    //参数校验
    if (templateUrl == null || templateUrl.isEmpty() || (!HttpUtil.isHttp(templateUrl) && !HttpUtil.isHttps(templateUrl))) {
      log.error("模板文件地址:{} 格式不正确!", templateUrl);
      throw new IllegalArgumentException("模板文件地址格式不正确!");
    }
    if (MapUtil.isEmpty(params)) {
      log.error("模板文件参数:{} 为空!", params);
      throw new IllegalArgumentException("模板文件参数为空!");
    }
    /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ word模板数据填充 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
    String timeStr = DateUtil.date().toString("yyyyMMddHHmmssSSS");
    File templateFile = new File(FileUtil.getTmpDir(), "Template_" + timeStr + ".docx");
    long l = HttpUtil.downloadFile(templateUrl, templateFile.getAbsolutePath());
    if (l == 0) {
      log.error("模板文件:{} 下载失败!", templateUrl);
      throw new IllegalArgumentException("模板文件下载失败!");
    }
    File wordFile = new File(FileUtil.getTmpDir(), "Word_" + timeStr + ".docx");
    try {
      ConfigureBuilder builder = Configure.builder();
      // 遍历给定的模板参数,如果存在预设的特定开头的key,则使用对应的预设策略进行渲染
      for (Map.Entry<String, Object> stringObjectEntry : params.entrySet()) {
        String key = stringObjectEntry.getKey();
        for (Map.Entry<String, Class<? extends RenderPolicy>> stringClassEntry : SPECIAL_KEY_MAP_CLASS.entrySet()) {
          String configureKey = stringClassEntry.getKey();
          if (key.startsWith(configureKey)) {
            Class<? extends RenderPolicy> value = stringClassEntry.getValue();
            RenderPolicy renderPolicy = ReflectUtil.newInstance(value);
            log.info("动态的绑定模板:{} 参数:{} 策略为:{}", templateUrl, key, renderPolicy.getClass().getName());
            builder.bind(key, renderPolicy);
            break;
          }
        }
      }
      //创建行循环策略
      Configure configure = builder.build();
      XWPFTemplate template = XWPFTemplate.compile(templateFile, configure)
              .render(params);
      //获取模板中所有的模板标签,用于和参数进行比对,给出提示,不做具体处理;
      List<MetaTemplate> elementTemplates = template.getElementTemplates();
      if (CollUtil.isNotEmpty(elementTemplates)) {
        Set<String> templateTags = elementTemplates.stream().filter(it -> it instanceof RunTemplate).map(it -> ((RunTemplate) it).getTagName()).collect(Collectors.toSet());
        if (templateTags.size() != params.size()) {
          Set<String> keys = params.keySet();
          //取templateTags中存在的,但是keys中不存在的元素
          Set<String> diff = new HashSet<>(templateTags);
          diff.removeAll(keys);
          log.error("模板:{} 中缺少如下字段值:{} 请检查传参!", templateUrl, JSON.toJSONString(diff));
        }
      }
      template.writeAndClose(new FileOutputStream(wordFile));
      PoitlIOUtils.closeQuietlyMulti(template);//关闭模板流
    } catch (Exception e) {
      log.error("word模板数据填充异常:{}", e.getMessage());
      throw new RuntimeException("word模板数据填充异常:" + e.getMessage());
    }
    /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ word模板数据填充 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */

    /* ↓↓↓↓↓↓ word转PDF ↓↓↓↓↓↓ */
    if (type == WORD_TRANS_TYPE.PDF) {
      try {
        File pdfFile = new File(FileUtil.getTmpDir(), "Pdf_" + timeStr + ".pdf");
        docxToPdf(wordFile, pdfFile);
        wordFile = pdfFile;
      } catch (Exception e) {
        log.error("word转pdf异常:{}", e.getMessage());
        throw new RuntimeException("word转pdf异常:" + e.getMessage());
      }
    }
    /* ↑↑↑↑↑↑ word转PDF ↑↑↑↑↑↑ */

    return wordFile.getAbsolutePath();
  }


  /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ word转pdf操作相关 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
  public static boolean getLicense() {
    boolean result = false;
    InputStream is = null;
    try {
      Resource resource = new ClassPathResource("license.xml"); // xml文件地址
      is = resource.getInputStream();
      License aposeLic = new License();
      aposeLic.setLicense(is);
      result = true;
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      if (is != null) {
        try {
          is.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    return result;
  }

  public static boolean docxToPdf(File inPath, File outPath, String... watermarkText) {
    return docxToPdf(inPath.getPath(), outPath.getPath(), watermarkText);
  }

  public static boolean docxToPdf(String inPath, String outPath, String... watermarkText) {
    // if (!getLicense()) { // 验证License 若不验证则转化出的pdf文档会有水印产生 //  破解版本的话,则不需要此;
    //     return false;
    // }
    FileOutputStream os = null;
    try {
      long old = System.currentTimeMillis();
      File file = new File(outPath); // 新建一个空白pdf文档
      os = new FileOutputStream(file);
      //默认中文的加载配置
      LoadOptions lo = new LoadOptions();
      lo.getLanguagePreferences().setDefaultEditingLanguage(EditingLanguage.CHINESE_PRC);
      Document document = new Document(inPath, lo); // Address是将要被转化的word文档
      TableCollection tables = document.getFirstSection().getBody().getTables();
      for (Table table : tables) {
        RowCollection rows = table.getRows();
        table.setAllowAutoFit(false);
        for (Row row : rows) {
          CellCollection cells = row.getCells();
          for (Cell cell : cells) {
            CellFormat cellFormat = cell.getCellFormat();
            cellFormat.setFitText(false);
            cellFormat.setWrapText(true);
          }
        }
      }
      //设置上边距，下边距. 上下边距设置成最小,防止出现跨页的情况;
      DocumentBuilder builder = new DocumentBuilder(document);
      builder.getPageSetup().setTopMargin(0);
      builder.getPageSetup().setBottomMargin(0);
      // builder.getPageSetup().setLeftMargin(1);
      // builder.getPageSetup().setRightMargin(1);
      document = builder.getDocument();

      //插入水印
      if (watermarkText.length > 0) {
        for (String waterMark : watermarkText) {
          insertWatermarkText(document, waterMark);
        }
      }

      //在使用中发现 apose 对 word 文档转换 PDF 操作中会出现将单页分成两页的情况。仔细分析后发现是因为 word 文档在编辑的时候是采用的多页编辑。页面效果是单页，可是在 apose 将 word 文档转为 pdf 后就变成了两页。所以要新生成一个 word 文档并保留原 word 文档的样式，问题解决。
      Document documentN = new Document();//新建一个空白pdf文档
      documentN.removeAllChildren();
      documentN.appendDocument(document, ImportFormatMode.USE_DESTINATION_STYLES);//保留样式

      PdfSaveOptions saveOptions = new PdfSaveOptions();
      // saveOptions.setExportDocumentStructure(true);//有文档结构的pdf//文档可以复制修改,不推荐
      // saveOptions.setEmbedFullFonts(true);//字体嵌入//文件会变的很大,不推荐
      // MetafileRenderingOptions metafileRenderingOptions = new MetafileRenderingOptions();
      // metafileRenderingOptions.setScaleWmfFontsToMetafileSize(false);
      // saveOptions.setMetafileRenderingOptions(metafileRenderingOptions);
      documentN.save(os, saveOptions);// 全面支持DOC, DOCX, OOXML, RTF HTML, OpenDocument, PDF,
// EPUB, XPS, SWF 相互转换
      long now = System.currentTimeMillis();
      System.out.println("pdf转换成功，共耗时：" + ((now - old) / 1000.0) + "秒"); // 转化用时
    } catch (Exception e) {
      e.printStackTrace();
      return false;
    } finally {
      if (os != null) {
        try {
          os.flush();
          os.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    return true;
  }

  /**
   * 为word文档添加水印
   *
   * @param doc           word文档模型
   * @param watermarkText 需要添加的水印字段
   * @throws Exception
   */
  public static void insertWatermarkText(Document doc, String watermarkText) throws Exception {
    Shape watermark = new Shape(doc, ShapeType.TEXT_PLAIN_TEXT);
    //水印内容
    watermark.getTextPath().setText(watermarkText);
    //水印字体
    TextPath textPath = watermark.getTextPath();
    textPath.setFontFamily("宋体");
    // textPath.setSize(16);
    //水印宽度
    watermark.setWidth(400);
    //水印高度
    watermark.setHeight(120);
    //旋转水印
    watermark.setRotation(-40);
    //水印颜色 浅灰色
    watermark.getFill().setColor(new Color(242, 242, 242));
    watermark.setStrokeColor(new Color(242, 242, 242));
    //设置相对水平位置
    watermark.setRelativeHorizontalPosition(RelativeHorizontalPosition.PAGE);
    //设置相对垂直位置
    watermark.setRelativeVerticalPosition(RelativeVerticalPosition.PAGE);
    //设置包装类型
    watermark.setWrapType(WrapType.NONE);
    //设置垂直对齐
    watermark.setVerticalAlignment(VerticalAlignment.CENTER);
    //设置文本水平对齐方式
    watermark.setHorizontalAlignment(HorizontalAlignment.CENTER);
    Paragraph watermarkPara = new Paragraph(doc);
    watermarkPara.appendChild(watermark);
    for (Section sect : doc.getSections()) {
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_PRIMARY);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_FIRST);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_EVEN);
    }
    System.out.println("Watermark Set");
  }

  /**
   * 在页眉中插入水印
   *
   * @param watermarkPara
   * @param sect
   * @param headerType
   * @throws Exception
   */
  private static void insertWatermarkIntoHeader(Paragraph watermarkPara, Section sect, int headerType) throws Exception {
    HeaderFooter header = sect.getHeadersFooters().getByHeaderFooterType(headerType);
    if (header == null) {
      header = new HeaderFooter(sect.getDocument(), headerType);
      sect.getHeadersFooters().add(header);
    }
    header.appendChild(watermarkPara.deepClone(true));
  }

  /**
   * 设置水印属性
   *
   * @param doc
   * @param wmText
   * @param left
   * @param top
   * @return
   * @throws Exception
   */
  public static Shape ShapeMore(Document doc, String wmText, double left, double top) throws Exception {
//        Shape waterShape = new Shape(doc, ShapeType.TEXT_PLAIN_TEXT);
    Shape waterShape = new Shape(doc, ShapeType.IMAGE);
    waterShape.getImageData().setImage(wmText);
    waterShape.setWidth(100.0);
    waterShape.setHeight(100.0);
    waterShape.setRotation(0);
    waterShape.setFilled(true);
//        //水印内容
//        waterShape.getTextPath().setText(wmText);
//        //水印字体
//        waterShape.getTextPath().setFontFamily("宋体");
//        //水印宽度
//        waterShape.setWidth(40);
//        //水印高度
//        waterShape.setHeight(13);
//        //旋转水印
//        waterShape.setRotation(-40);
    //水印颜色 浅灰色
        /*waterShape.getFill().setColor(Color.lightGray);
        waterShape.setStrokeColor(Color.lightGray);*/
    waterShape.setStrokeColor(new Color(210, 210, 210));
    //将水印放置在页面中心
    waterShape.setLeft(left);
    waterShape.setTop(top);
    //设置包装类型
    waterShape.setWrapType(WrapType.NONE);
    return waterShape;
  }

  /**
   * 插入多个水印
   *
   * @param mdoc
   * @param wmText
   * @throws Exception
   */
  public static void WaterMarkMore(Document mdoc, String wmText) throws Exception {
    Paragraph watermarkPara = new Paragraph(mdoc);
//        for (int j = 0; j < 500; j = j + 100)
//        {
//            for (int i = 0; i < 700; i = i + 85)
//            {
//                Shape waterShape = ShapeMore(mdoc, wmText, j, i);
//                watermarkPara.appendChild(waterShape);
//            }
//        }
    Shape waterShape = ShapeMore(mdoc, wmText, 155, 300);
    watermarkPara.appendChild(waterShape);
    for (Section sect : mdoc.getSections()) {
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_PRIMARY);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_FIRST);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_EVEN);
    }
  }
  /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ word转pdf操作相关 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */
}
import cn.hutool.core.collection.CollUtil;
        import cn.hutool.core.date.DateUtil;
        import cn.hutool.core.io.FileUtil;
        import cn.hutool.core.map.MapUtil;
        import cn.hutool.core.util.ReflectUtil;
        import cn.hutool.http.HttpUtil;
        import com.alibaba.fastjson.JSON;
        import com.aspose.words.Shape;
        import com.aspose.words.*;
        import com.deepoove.poi.XWPFTemplate;
        import com.deepoove.poi.config.Configure;
        import com.deepoove.poi.config.ConfigureBuilder;
        import com.deepoove.poi.plugin.table.LoopRowTableRenderPolicy;
        import com.deepoove.poi.policy.RenderPolicy;
        import com.deepoove.poi.template.MetaTemplate;
        import com.deepoove.poi.template.run.RunTemplate;
        import com.deepoove.poi.util.PoitlIOUtils;
        import lombok.extern.slf4j.Slf4j;
        import org.springframework.core.io.ClassPathResource;
        import org.springframework.core.io.Resource;

        import java.awt.*;
        import java.io.File;
        import java.io.FileOutputStream;
        import java.io.IOException;
        import java.io.InputStream;
        import java.util.*;
        import java.util.List;
        import java.util.stream.Collectors;

/**
 * @description word, pdf操作工具类
 **/
@Slf4j
public class WordPdfUtil {

  public static void main(String[] args) {
    //测试
    //模板1:
    String templateName = "1.机械设备买卖合同范本修订（修订）.docx";
    String templateUrl = "https://stage-minio1-view-dic.diccp.com/ctcemti-ggc-default/word_template_jx_1.docx";
    HashMap<String, Object> params = new HashMap<String, Object>() {{
      put("number", "1123");
      put("name", "测试工程");
      put("buy", "张三");
      put("sale", "天恩科技");
      put("location", "北京市海淀区");
      put("year", "2025");
      put("month", "9");
      put("day", "9");
      put("LoopRow_Devices", new ArrayList<Map<String, Object>>() {{
        add(new HashMap<String, Object>() {{
          put("name", "设备1");
          put("xh", "型号xx");
          put("unit", "个");
          put("num", "1");
          put("noTaxPrice", "11");
          put("hasTaxPrice", "22");
          put("noTaxAll", "33");
          put("factory", "天恩");
          put("remark", "无备注");
        }});
        add(new HashMap<String, Object>() {{
          put("name", "设备2");
          put("xh", "型号2xx");
          put("unit", "个");
          put("num", "2");
          put("noTaxPrice", "22");
          put("hasTaxPrice", "33");
          put("noTaxAll", "55");
          put("factory", "天恩2");
          put("remark", "无备注1");
        }});
      }});
      put("zzTax", "123");
      put("hasTaxAll", "456");
      //小计类
      put("xhXj", "11");
      put("unitXj", "22");
      put("numXj", "33");
      put("noTaxPriceXj", "44");
      put("hasTaxPriceXj", "55");
      put("noTaxAllXj", "66");
    }};
    String fileUrl = inflateWordTemplate(templateUrl, params, WORD_TRANS_TYPE.WORD);
    log.info("模板:{} 文件路径:{}", templateName, fileUrl);
    fileUrl = inflateWordTemplate(templateUrl, params, WORD_TRANS_TYPE.PDF);
    log.info("模板:{} 文件路径:{}", templateName, fileUrl);


  }

  public enum WORD_TRANS_TYPE {
    WORD, PDF
  }

  /**
   * @description 特殊的渲染策略的key前缀和对应的渲染策略类, TODO 如果有特殊的格式类型,需要在此处进行追加;
   **/
  public static final Map<String, Class<? extends RenderPolicy>> SPECIAL_KEY_MAP_CLASS = new HashMap<String, Class<? extends RenderPolicy>>() {{
    put("LoopRow", LoopRowTableRenderPolicy.class);
  }};

  /**
   * @description: 填充word模板文件参数
   * @Param templateUrl: 模板文件地址
   * @Param params: 模板文件参数
   * @Param type: 输出文件类型枚举,WORD,PDF
   * @return: java.lang.String 填充了模板参数的文件路径
   **/
  public static String inflateWordTemplate(String templateUrl, Map<String, Object> params, WORD_TRANS_TYPE type) {
    //参数校验
    if (templateUrl == null || templateUrl.isEmpty() || (!HttpUtil.isHttp(templateUrl) && !HttpUtil.isHttps(templateUrl))) {
      log.error("模板文件地址:{} 格式不正确!", templateUrl);
      throw new IllegalArgumentException("模板文件地址格式不正确!");
    }
    if (MapUtil.isEmpty(params)) {
      log.error("模板文件参数:{} 为空!", params);
      throw new IllegalArgumentException("模板文件参数为空!");
    }
    /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ word模板数据填充 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
    String timeStr = DateUtil.date().toString("yyyyMMddHHmmssSSS");
    File templateFile = new File(FileUtil.getTmpDir(), "Template_" + timeStr + ".docx");
    long l = HttpUtil.downloadFile(templateUrl, templateFile.getAbsolutePath());
    if (l == 0) {
      log.error("模板文件:{} 下载失败!", templateUrl);
      throw new IllegalArgumentException("模板文件下载失败!");
    }
    File wordFile = new File(FileUtil.getTmpDir(), "Word_" + timeStr + ".docx");
    try {
      ConfigureBuilder builder = Configure.builder();
      // 遍历给定的模板参数,如果存在预设的特定开头的key,则使用对应的预设策略进行渲染
      for (Map.Entry<String, Object> stringObjectEntry : params.entrySet()) {
        String key = stringObjectEntry.getKey();
        for (Map.Entry<String, Class<? extends RenderPolicy>> stringClassEntry : SPECIAL_KEY_MAP_CLASS.entrySet()) {
          String configureKey = stringClassEntry.getKey();
          if (key.startsWith(configureKey)) {
            Class<? extends RenderPolicy> value = stringClassEntry.getValue();
            RenderPolicy renderPolicy = ReflectUtil.newInstance(value);
            log.info("动态的绑定模板:{} 参数:{} 策略为:{}", templateUrl, key, renderPolicy.getClass().getName());
            builder.bind(key, renderPolicy);
            break;
          }
        }
      }
      //创建行循环策略
      Configure configure = builder.build();
      XWPFTemplate template = XWPFTemplate.compile(templateFile, configure)
              .render(params);
      //获取模板中所有的模板标签,用于和参数进行比对,给出提示,不做具体处理;
      List<MetaTemplate> elementTemplates = template.getElementTemplates();
      if (CollUtil.isNotEmpty(elementTemplates)) {
        Set<String> templateTags = elementTemplates.stream().filter(it -> it instanceof RunTemplate).map(it -> ((RunTemplate) it).getTagName()).collect(Collectors.toSet());
        if (templateTags.size() != params.size()) {
          Set<String> keys = params.keySet();
          //取templateTags中存在的,但是keys中不存在的元素
          Set<String> diff = new HashSet<>(templateTags);
          diff.removeAll(keys);
          log.error("模板:{} 中缺少如下字段值:{} 请检查传参!", templateUrl, JSON.toJSONString(diff));
        }
      }
      template.writeAndClose(new FileOutputStream(wordFile));
      PoitlIOUtils.closeQuietlyMulti(template);//关闭模板流
    } catch (Exception e) {
      log.error("word模板数据填充异常:{}", e.getMessage());
      throw new RuntimeException("word模板数据填充异常:" + e.getMessage());
    }
    /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ word模板数据填充 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */

    /* ↓↓↓↓↓↓ word转PDF ↓↓↓↓↓↓ */
    if (type == WORD_TRANS_TYPE.PDF) {
      try {
        File pdfFile = new File(FileUtil.getTmpDir(), "Pdf_" + timeStr + ".pdf");
        docxToPdf(wordFile, pdfFile);
        wordFile = pdfFile;
      } catch (Exception e) {
        log.error("word转pdf异常:{}", e.getMessage());
        throw new RuntimeException("word转pdf异常:" + e.getMessage());
      }
    }
    /* ↑↑↑↑↑↑ word转PDF ↑↑↑↑↑↑ */

    return wordFile.getAbsolutePath();
  }


  /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ word转pdf操作相关 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
  public static boolean getLicense() {
    boolean result = false;
    InputStream is = null;
    try {
      Resource resource = new ClassPathResource("license.xml"); // xml文件地址
      is = resource.getInputStream();
      License aposeLic = new License();
      aposeLic.setLicense(is);
      result = true;
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
      if (is != null) {
        try {
          is.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    return result;
  }

  public static boolean docxToPdf(File inPath, File outPath, String... watermarkText) {
    return docxToPdf(inPath.getPath(), outPath.getPath(), watermarkText);
  }

  public static boolean docxToPdf(String inPath, String outPath, String... watermarkText) {
    // if (!getLicense()) { // 验证License 若不验证则转化出的pdf文档会有水印产生 //  破解版本的话,则不需要此;
    //     return false;
    // }
    FileOutputStream os = null;
    try {
      long old = System.currentTimeMillis();
      File file = new File(outPath); // 新建一个空白pdf文档
      os = new FileOutputStream(file);
      //默认中文的加载配置
      LoadOptions lo = new LoadOptions();
      lo.getLanguagePreferences().setDefaultEditingLanguage(EditingLanguage.CHINESE_PRC);
      Document document = new Document(inPath, lo); // Address是将要被转化的word文档
      TableCollection tables = document.getFirstSection().getBody().getTables();
      for (Table table : tables) {
        RowCollection rows = table.getRows();
        table.setAllowAutoFit(false);
        for (Row row : rows) {
          CellCollection cells = row.getCells();
          for (Cell cell : cells) {
            CellFormat cellFormat = cell.getCellFormat();
            cellFormat.setFitText(false);
            cellFormat.setWrapText(true);
          }
        }
      }
      //设置上边距，下边距. 上下边距设置成最小,防止出现跨页的情况;
      DocumentBuilder builder = new DocumentBuilder(document);
      builder.getPageSetup().setTopMargin(0);
      builder.getPageSetup().setBottomMargin(0);
      // builder.getPageSetup().setLeftMargin(1);
      // builder.getPageSetup().setRightMargin(1);
      document = builder.getDocument();

      //插入水印
      if (watermarkText.length > 0) {
        for (String waterMark : watermarkText) {
          insertWatermarkText(document, waterMark);
        }
      }

      //在使用中发现 apose 对 word 文档转换 PDF 操作中会出现将单页分成两页的情况。仔细分析后发现是因为 word 文档在编辑的时候是采用的多页编辑。页面效果是单页，可是在 apose 将 word 文档转为 pdf 后就变成了两页。所以要新生成一个 word 文档并保留原 word 文档的样式，问题解决。
      Document documentN = new Document();//新建一个空白pdf文档
      documentN.removeAllChildren();
      documentN.appendDocument(document, ImportFormatMode.USE_DESTINATION_STYLES);//保留样式

      PdfSaveOptions saveOptions = new PdfSaveOptions();
      // saveOptions.setExportDocumentStructure(true);//有文档结构的pdf//文档可以复制修改,不推荐
      // saveOptions.setEmbedFullFonts(true);//字体嵌入//文件会变的很大,不推荐
      // MetafileRenderingOptions metafileRenderingOptions = new MetafileRenderingOptions();
      // metafileRenderingOptions.setScaleWmfFontsToMetafileSize(false);
      // saveOptions.setMetafileRenderingOptions(metafileRenderingOptions);
      documentN.save(os, saveOptions);// 全面支持DOC, DOCX, OOXML, RTF HTML, OpenDocument, PDF,
// EPUB, XPS, SWF 相互转换
      long now = System.currentTimeMillis();
      System.out.println("pdf转换成功，共耗时：" + ((now - old) / 1000.0) + "秒"); // 转化用时
    } catch (Exception e) {
      e.printStackTrace();
      return false;
    } finally {
      if (os != null) {
        try {
          os.flush();
          os.close();
        } catch (IOException e) {
          e.printStackTrace();
        }
      }
    }
    return true;
  }

  /**
   * 为word文档添加水印
   *
   * @param doc           word文档模型
   * @param watermarkText 需要添加的水印字段
   * @throws Exception
   */
  public static void insertWatermarkText(Document doc, String watermarkText) throws Exception {
    Shape watermark = new Shape(doc, ShapeType.TEXT_PLAIN_TEXT);
    //水印内容
    watermark.getTextPath().setText(watermarkText);
    //水印字体
    TextPath textPath = watermark.getTextPath();
    textPath.setFontFamily("宋体");
    // textPath.setSize(16);
    //水印宽度
    watermark.setWidth(400);
    //水印高度
    watermark.setHeight(120);
    //旋转水印
    watermark.setRotation(-40);
    //水印颜色 浅灰色
    watermark.getFill().setColor(new Color(242, 242, 242));
    watermark.setStrokeColor(new Color(242, 242, 242));
    //设置相对水平位置
    watermark.setRelativeHorizontalPosition(RelativeHorizontalPosition.PAGE);
    //设置相对垂直位置
    watermark.setRelativeVerticalPosition(RelativeVerticalPosition.PAGE);
    //设置包装类型
    watermark.setWrapType(WrapType.NONE);
    //设置垂直对齐
    watermark.setVerticalAlignment(VerticalAlignment.CENTER);
    //设置文本水平对齐方式
    watermark.setHorizontalAlignment(HorizontalAlignment.CENTER);
    Paragraph watermarkPara = new Paragraph(doc);
    watermarkPara.appendChild(watermark);
    for (Section sect : doc.getSections()) {
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_PRIMARY);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_FIRST);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_EVEN);
    }
    System.out.println("Watermark Set");
  }

  /**
   * 在页眉中插入水印
   *
   * @param watermarkPara
   * @param sect
   * @param headerType
   * @throws Exception
   */
  private static void insertWatermarkIntoHeader(Paragraph watermarkPara, Section sect, int headerType) throws Exception {
    HeaderFooter header = sect.getHeadersFooters().getByHeaderFooterType(headerType);
    if (header == null) {
      header = new HeaderFooter(sect.getDocument(), headerType);
      sect.getHeadersFooters().add(header);
    }
    header.appendChild(watermarkPara.deepClone(true));
  }

  /**
   * 设置水印属性
   *
   * @param doc
   * @param wmText
   * @param left
   * @param top
   * @return
   * @throws Exception
   */
  public static Shape ShapeMore(Document doc, String wmText, double left, double top) throws Exception {
//        Shape waterShape = new Shape(doc, ShapeType.TEXT_PLAIN_TEXT);
    Shape waterShape = new Shape(doc, ShapeType.IMAGE);
    waterShape.getImageData().setImage(wmText);
    waterShape.setWidth(100.0);
    waterShape.setHeight(100.0);
    waterShape.setRotation(0);
    waterShape.setFilled(true);
//        //水印内容
//        waterShape.getTextPath().setText(wmText);
//        //水印字体
//        waterShape.getTextPath().setFontFamily("宋体");
//        //水印宽度
//        waterShape.setWidth(40);
//        //水印高度
//        waterShape.setHeight(13);
//        //旋转水印
//        waterShape.setRotation(-40);
    //水印颜色 浅灰色
        /*waterShape.getFill().setColor(Color.lightGray);
        waterShape.setStrokeColor(Color.lightGray);*/
    waterShape.setStrokeColor(new Color(210, 210, 210));
    //将水印放置在页面中心
    waterShape.setLeft(left);
    waterShape.setTop(top);
    //设置包装类型
    waterShape.setWrapType(WrapType.NONE);
    return waterShape;
  }

  /**
   * 插入多个水印
   *
   * @param mdoc
   * @param wmText
   * @throws Exception
   */
  public static void WaterMarkMore(Document mdoc, String wmText) throws Exception {
    Paragraph watermarkPara = new Paragraph(mdoc);
//        for (int j = 0; j < 500; j = j + 100)
//        {
//            for (int i = 0; i < 700; i = i + 85)
//            {
//                Shape waterShape = ShapeMore(mdoc, wmText, j, i);
//                watermarkPara.appendChild(waterShape);
//            }
//        }
    Shape waterShape = ShapeMore(mdoc, wmText, 155, 300);
    watermarkPara.appendChild(waterShape);
    for (Section sect : mdoc.getSections()) {
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_PRIMARY);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_FIRST);
      insertWatermarkIntoHeader(watermarkPara, sect, HeaderFooterType.HEADER_EVEN);
    }
  }
  /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ word转pdf操作相关 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */
}
```

#### 3.使用方法
```java
String templateUrl = "https://stage-minio1-view-dic.diccp.com/ctcemti-ggc-default/word_template_jx_1.docx";//文件地址
HashMap<String, Object> params = new HashMap<String, Object>() {{
    put("number", "1123");//普通的模板文本标记
    put("name", "测试工程");
    put("buy", "张三");
    put("sale", "天恩科技");
    put("location", "北京市海淀区");
    put("year", "2025");
    put("month", "9");
    put("day", "9");
    put("LoopRow_Devices", new ArrayList<Map<String, Object>>() {{//word中嵌套了需要循环行的表格,使用固定的LoopRow进行开头,用于工具类后续的判断
        add(new HashMap<String, Object>() {{
            put("name", "设备1");
            put("xh", "型号xx");
            put("unit", "个");
            put("num", "1");
            put("noTaxPrice", "11");
            put("hasTaxPrice", "22");
            put("noTaxAll", "33");
            put("factory", "天恩");
            put("remark", "无备注");
        }});
        add(new HashMap<String, Object>() {{
            put("name", "设备2");
            put("xh", "型号2xx");
            put("unit", "个");
            put("num", "2");
            put("noTaxPrice", "22");
            put("hasTaxPrice", "33");
            put("noTaxAll", "55");
            put("factory", "天恩2");
            put("remark", "无备注1");
        }});
    }});
    put("zzTax", "123");
    put("hasTaxAll", "456");
    //小计类
    put("xhXj", "11");
    put("unitXj", "22");
    put("numXj", "33");
    put("noTaxPriceXj", "44");
    put("hasTaxPriceXj", "55");
    put("noTaxAllXj", "66");
    put("img", "http://www.baidu.com/img/bdlogo.png");//图片类型直接的传图片的地址
}};//模板内容
// String filePath = WordPdfUtil.inflateWordTemplate(templateUrl, params, WordPdfUtil.WORD_TRANS_TYPE.PDF);
// setFileName(response, "设备台账.pdf");
String filePath = WordPdfUtil.inflateWordTemplate(templateUrl, params, WordPdfUtil.WORD_TRANS_TYPE.WORD);
setFileName(response, "设备台账.docx");
//写回到响应流中
OutputStream out = response.getOutputStream();
BufferedOutputStream bos = new BufferedOutputStream(out);
IoUtil.copy(FileUtil.getInputStream(filePath),bos);
bos.flush();
out.flush();
PoitlIOUtils.closeQuietlyMulti(bos, out);
```