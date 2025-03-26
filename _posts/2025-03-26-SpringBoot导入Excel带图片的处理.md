---
title: SpringBoot导入Excel带图片的处理
description: 导入Excel中带图片的话，通常不好处理。此处结合实际给出一种解决方案，但是此种方案存在限制，就是Excel中的图片必须是WPS的嵌入式图片，否则无法进行读取；
date: 2025-03-26 17:32:00 +0800
categories: [后端, 高级操作]
tags: [SpringBoot,文件导入,POI]
toc: true
comments: true
mermaid: true
math: true
pin: false
image:
  path: /assets/img/posts/poi.png
  alt: SpringBoot导入Excel带图片的处理
---

##### 一、工具类
1. 核心工具类（依赖hutool,lombok）
```java

import cn.afterturn.easypoi.excel.annotation.Excel;
import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.collection.CollectionUtil;
import cn.hutool.core.io.FileUtil;
import cn.hutool.core.io.IoUtil;
import cn.hutool.core.lang.Assert;
import cn.hutool.core.util.*;
import cn.hutool.json.JSONArray;
import cn.hutool.json.JSONObject;
import cn.hutool.json.XML;
import cn.hutool.poi.excel.ExcelReader;
import cn.hutool.poi.excel.ExcelUtil;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.collections4.map.HashedMap;
import org.apache.commons.io.IOUtils;
import org.apache.poi.hssf.usermodel.*;
import org.apache.poi.ooxml.POIXMLDocumentPart;
import org.apache.poi.openxml4j.opc.PackagePartName;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.*;
import org.openxmlformats.schemas.drawingml.x2006.spreadsheetDrawing.CTMarker;
import org.springframework.web.multipart.MultipartFile;
import org.tekj.base.utils.lambda.LambdaUtils;

import java.io.*;
import java.lang.reflect.Field;
import java.nio.charset.StandardCharsets;
import java.util.*;
import java.util.zip.ZipEntry;
import java.util.zip.ZipInputStream;

/**
 * Excel图片工具类
 */
@Slf4j
public class ExcelPicUtilPro {

    /**
     * 解析excel中的嵌入的图片(必须使用WPS的嵌入图片功能), 将图片放到本地的临时目录中, 并将图片的路径映射到对应的字段中
     *
     * @date: 2025/3/26 15:36
     * @Param file: 上传的excel文件
     * @Param sheetIndex: excel的sheet索引,从0开始计数,0表示第一个sheet
     * @Param headerRowIndex: 表头的行数,从0开始计数,0表示第一行
     * @Param distList: 需要填充的对象列表
     * @Param imgFileFieldNames: 图片字段的反射集合,至少一个
     **/
    public static <T, R> void flatExcelImg2List(MultipartFile file,
                                                Integer headerRowIndex,
                                                List<T> distList,
                                                LambdaUtils.SFunction<T, R>... imgFileFieldNames) {
        flatExcelImg2List(file, 0, headerRowIndex, distList, imgFileFieldNames);
    }

    @SneakyThrows
    public static <T, R> void flatExcelImg2List(MultipartFile file,
                                                Integer sheetIndex,
                                                Integer headerRowIndex,
                                                List<T> distList,
                                                LambdaUtils.SFunction<T, R>... imgFileFieldNames) {
        if (file == null || file.isEmpty()) {
            log.error("上传的excel文件不能为空");
            return;
        }
        if (CollUtil.isEmpty(distList)) {
            log.error("填充excel中的图片文件,distList不能为空");
            return;
        }
        if (imgFileFieldNames.length < 1) {
            log.error("至少需要一个图片字段");
            return;
        }
        if (sheetIndex == null) {
            sheetIndex = 0;
        }
        if (headerRowIndex == null) {
            headerRowIndex = 0;
        }

        // 解析excel文件
        ExcelReader reader = ExcelUtil.getReader(file.getInputStream());
        Row row = reader.getWorkbook().getSheetAt(sheetIndex).getRow(headerRowIndex);
        short firstCellNum = row.getFirstCellNum();
        short lastCellNum = row.getLastCellNum();
        Map<String, PictureData> PIC_MAP = getPicMap(file, reader.getWorkbook(), sheetIndex);

        List<LambdaUtils.SFunction<T, R>> fileFieldNameList = Arrays.asList(imgFileFieldNames);
        for (LambdaUtils.SFunction<T, R> trsFunction : fileFieldNameList) {
            LambdaUtils.LambdaMeta meta = LambdaUtils.extract(trsFunction);
            String implMethodName = meta.getImplMethodName();
            String fileFieldName = LambdaUtils.methodToProperty(implMethodName);
            Class<?> instantiatedClass = meta.getInstantiatedClass();
            // 图片在excel中的列索引
            int imgColIndex = -1;
            //获取字段的@Excel注解中的name名称
            Field imgFiled = ReflectUtil.getField(instantiatedClass, fileFieldName);
            Excel annotation = imgFiled.getAnnotation(Excel.class);
            String excelName = StrUtil.trim(annotation.name());
            for (int i = firstCellNum; i < lastCellNum; i++) {
                String cellValue = row.getCell(i).getStringCellValue();
                if (StrUtil.equals(StrUtil.trim(cellValue), excelName)) {
                    imgColIndex = i;
                    break;
                }
            }
            if (imgColIndex == -1) {
                log.error("获取excel中的图片文件,在给定字段:{} 的@Excel注解中,没有找到对应的列名:{}", fileFieldName, excelName);
                continue;
            }

            for (int rowIdx = 0; rowIdx < distList.size(); rowIdx++) {
                Object o = distList.get(rowIdx);
                if (!ReflectUtil.hasField(o.getClass(), fileFieldName)) {
                    log.error("填充excel中的图片文件不存在,请检查字段名是否正确,字段名:{} 不存在", fileFieldName);
                    break;
                }
                String key = (rowIdx + 1 + headerRowIndex) + "_" + imgColIndex;
                if (PIC_MAP.containsKey(key)) {
                    PictureData pictureData = PIC_MAP.get(key);
                    if (pictureData != null) {
                        String mimeType = pictureData.getMimeType();
                        File imgFile = new File(FileUtil.getTmpDir(), IdUtil.simpleUUID() + "." + mimeType.split("/")[1]);
                        FileUtil.writeBytes(pictureData.getData(), imgFile);
                        log.info("excel临时图片:第{}行,第{}列, {},图片路径：{}", (rowIdx + 1 + headerRowIndex), imgColIndex, fileFieldName, imgFile.getAbsolutePath());
                        if (imgFiled.getType() == String.class) {
                            ReflectUtil.setFieldValue(o, fileFieldName, imgFile.getAbsolutePath());
                        }
                        if (imgFiled.getType() == File.class) {
                            ReflectUtil.setFieldValue(o, fileFieldName, imgFile);
                        }
                    }
                }
            }
        }
    }

    /**
     * 获取工作簿指定sheet中图片列表
     *
     * @param workbook   工作簿{@link Workbook}
     * @param sheetIndex sheet的索引
     * @return 图片映射，键格式：行_列，值：{@link PictureData}
     */
    public static Map<String, PictureData> getPicMap(MultipartFile file, Workbook workbook, int sheetIndex) {
        Assert.notNull(workbook, "Workbook must be not null !");
        if (sheetIndex < 0) {
            sheetIndex = 0;
        }

        if (workbook instanceof HSSFWorkbook) {
            return getPicMapXls((HSSFWorkbook) workbook, sheetIndex);
        } else if (workbook instanceof XSSFWorkbook) {
            return getPicMapXlsx(file, (XSSFWorkbook) workbook, sheetIndex);
        } else {
            throw new IllegalArgumentException(StrUtil.format("Workbook type [{}] is not supported!", workbook.getClass()));
        }
    }

    // -------------------------------------------------------------------------------------------------------------- Private method start

    /**
     * 获取XLS工作簿指定sheet中图片列表
     *
     * @param workbook   工作簿{@link Workbook}
     * @param sheetIndex sheet的索引
     * @return 图片映射，键格式：行_列，值：{@link PictureData}
     */
    private static Map<String, PictureData> getPicMapXls(HSSFWorkbook workbook, int sheetIndex) {
        final Map<String, PictureData> picMap = new HashMap<>();
        final List<HSSFPictureData> pictures = workbook.getAllPictures();
        if (CollectionUtil.isNotEmpty(pictures)) {
            final HSSFSheet sheet = workbook.getSheetAt(sheetIndex);
            HSSFClientAnchor anchor;
            int pictureIndex;
            for (HSSFShape shape : sheet.getDrawingPatriarch().getChildren()) {
                if (shape instanceof HSSFPicture) {
                    pictureIndex = ((HSSFPicture) shape).getPictureIndex() - 1;
                    anchor = (HSSFClientAnchor) shape.getAnchor();
                    picMap.put(StrUtil.format("{}_{}", anchor.getRow1(), anchor.getCol1()), pictures.get(pictureIndex));
                }
            }
        }
        return picMap;
    }

    /**
     * 获取XLSX工作簿指定sheet中图片列表
     *
     * @param workbook   工作簿{@link Workbook}
     * @param sheetIndex sheet的索引
     * @return 图片映射，键格式：行_列，值：{@link PictureData}
     */
    private static Map<String, PictureData> getPicMapXlsx(MultipartFile file, XSSFWorkbook workbook, int sheetIndex) {
        Map<String, PictureData> sheetIndexPicMap = new HashMap<>();
        //映射对应的索引
        final XSSFSheet sheet = workbook.getSheetAt(sheetIndex);
        try {
            //wps图片读取,加索引
            //图片id对应图片文件的映射
            Map<String, XSSFPictureData> idMap = getPictures(IoUtil.readBytes(file.getInputStream()));
            for (Row row : sheet) {
                for (Cell cell : row) {
                    if (cell.getCellType() == CellType.FORMULA) {
                        String value = cell.getStringCellValue();
                        if (StrUtil.startWith(value, "=DISPIMG")) {//wps图片
                            String regex = "=DISPIMG\\(\"(.*)\",\\d{1}\\)";
                            String ID = ReUtil.getGroup1(regex, value);
                            log.info("excel图片ID:{}", ID);
                            if (StrUtil.isNotEmpty(ID) && idMap.containsKey(ID)) {
                                sheetIndexPicMap.put(StrUtil.format("{}_{}", cell.getRowIndex(), cell.getColumnIndex()), idMap.get(ID));
                            }
                        }
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        //标准的xlsx图片读取
        XSSFDrawing drawing;
        for (POIXMLDocumentPart dr : sheet.getRelations()) {
            if (dr instanceof XSSFDrawing) {
                drawing = (XSSFDrawing) dr;
                final List<XSSFShape> shapes = drawing.getShapes();
                XSSFPicture pic;
                CTMarker ctMarker;
                for (XSSFShape shape : shapes) {
                    if (shape instanceof XSSFPicture) {
                        pic = (XSSFPicture) shape;
                        ctMarker = pic.getPreferredSize().getFrom();
                        sheetIndexPicMap.put(StrUtil.format("{}_{}", ctMarker.getRow(), ctMarker.getCol()), pic.getPictureData());
                    }
                    // 其他类似于图表等忽略，see: https://gitee.com/dromara/hutool/issues/I38857
                }
            }
        }
        return sheetIndexPicMap;
    }
    // -------------------------------------------------------------------------------------------------------------- Private method end


    /**
     * 获取 WPS 文档中的图片，包括嵌入式图片和浮动式图片。
     *
     * @param data 二进制数据
     * @return 图片信息的 map
     * @throws IOException
     */
    public static Map<String, XSSFPictureData> getPictures(byte[] data) {
        try {
            Map<String, String> mapConfig = processZipEntries(new ByteArrayInputStream(data));
            Map<String, XSSFPictureData> mapPictures = processPictures(new ByteArrayInputStream(data), mapConfig);
            Iterator<Sheet> sheetIterator = WorkbookFactory.create(new ByteArrayInputStream(data)).sheetIterator();
            while (sheetIterator.hasNext()) {
                mapPictures.putAll(getFloatingPictures((XSSFSheet) sheetIterator.next()));
            }
            return mapPictures;
        } catch (IOException e) {
            return new HashedMap<>();
        }
    }

    /**
     * 获取浮动图片，以 map 形式返回，键为行列格式 x-y。
     *
     * @param xssfSheet WPS 工作表
     * @return 浮动图片的 map
     */
    public static Map<String, XSSFPictureData> getFloatingPictures(XSSFSheet xssfSheet) {
        Map<String, XSSFPictureData> mapFloatingPictures = new HashMap<>();
        XSSFDrawing drawingPatriarch = xssfSheet.getDrawingPatriarch();
        if (drawingPatriarch != null) {
            List<XSSFShape> shapes = drawingPatriarch.getShapes();
            for (XSSFShape shape : shapes) {
                if (shape instanceof XSSFPicture) {
                    XSSFClientAnchor anchor = (XSSFClientAnchor) shape.getAnchor();
                    XSSFPictureData pictureData = ((XSSFPicture) shape).getPictureData();
                    String key = anchor.getRow1() + "-" + anchor.getCol1();
                    mapFloatingPictures.put(key, pictureData);
                }
            }
        }
        return mapFloatingPictures;
    }

    /**
     * 处理 WPS 文件中的图片数据，返回图片信息 map。
     *
     * @param stream    输入流
     * @param mapConfig 配置映射
     * @return 图片信息的 map
     * @throws IOException
     */
    private static Map<String, XSSFPictureData> processPictures(ByteArrayInputStream
                                                                        stream, Map<String, String> mapConfig) throws IOException {
        Map<String, XSSFPictureData> mapPictures = new HashedMap<>();
        Workbook workbook = WorkbookFactory.create(stream);
        List<XSSFPictureData> allPictures = (List<XSSFPictureData>) workbook.getAllPictures();
        for (XSSFPictureData pictureData : allPictures) {
            PackagePartName partName = pictureData.getPackagePart().getPartName();
            String uri = partName.getURI().toString();
            if (mapConfig.containsKey(uri)) {
                String strId = mapConfig.get(uri);
                mapPictures.put(strId, pictureData);
            }
        }
        return mapPictures;
    }

    /**
     * 处理 Zip 文件中的条目，更新图片配置信息。
     *
     * @param stream Zip 输入流
     * @return 配置信息的 map
     * @throws IOException
     */
    private static Map<String, String> processZipEntries(ByteArrayInputStream stream) throws IOException {
        Map<String, String> mapConfig = new HashedMap<>();
        ZipInputStream zipInputStream = new ZipInputStream(stream);
        ZipEntry zipEntry;
        while ((zipEntry = zipInputStream.getNextEntry()) != null) {
            try {
                final String fileName = zipEntry.getName();
                if ("xl/cellimages.xml".equals(fileName)) {
                    processCellImages(zipInputStream, mapConfig);
                } else if ("xl/_rels/cellimages.xml.rels".equals(fileName)) {
                    return processCellImagesRels(zipInputStream, mapConfig);
                }
            } finally {
                zipInputStream.closeEntry();
            }
        }
        return new HashedMap<>();
    }

    /**
     * 处理 Zip 文件中的 cellimages.xml 文件，更新图片配置信息。
     *
     * @param zipInputStream Zip 输入流
     * @param mapConfig      配置信息的 map
     * @throws IOException
     */
    private static void processCellImages(ZipInputStream zipInputStream, Map<String, String> mapConfig) throws
            IOException {
        String content = IOUtils.toString(zipInputStream, StandardCharsets.UTF_8);
        JSONObject jsonObject = XML.toJSONObject(content);
        if (jsonObject != null) {
            JSONObject cellImages = jsonObject.getJSONObject("etc:cellImages");
            if (cellImages != null) {
                JSONArray cellImageArray = null;
                Object cellImage = cellImages.get("etc:cellImage");
                if (cellImage != null && cellImage instanceof JSONArray) {
                    cellImageArray = (JSONArray) cellImage;
                } else if (cellImage != null && cellImage instanceof JSONObject) {
                    cellImageArray = new JSONArray();
                    cellImageArray.add(cellImage);
                }
                if (cellImageArray != null) {
                    processImageItems(cellImageArray, mapConfig);
                }
            }
        }
    }

    /**
     * 处理 cellImageArray 中的图片项，更新图片配置信息。
     *
     * @param cellImageArray 图片项的 JSONArray
     * @param mapConfig      配置信息的 map
     */
    private static void processImageItems(JSONArray cellImageArray, Map<String, String> mapConfig) {
        for (int i = 0; i < cellImageArray.size(); i++) {
            JSONObject imageItem = cellImageArray.getJSONObject(i);
            if (imageItem != null) {
                JSONObject pic = imageItem.getJSONObject("xdr:pic");
                if (pic != null) {
                    processPic(pic, mapConfig);
                }
            }
        }
    }

    /**
     * 处理 pic 中的图片信息，更新图片配置信息。
     *
     * @param pic       图片的 JSONObject
     * @param mapConfig 配置信息的 map
     */
    private static void processPic(JSONObject pic, Map<String, String> mapConfig) {
        JSONObject nvPicPr = pic.getJSONObject("xdr:nvPicPr");
        if (nvPicPr != null) {
            JSONObject cNvPr = nvPicPr.getJSONObject("xdr:cNvPr");
            if (cNvPr != null) {
                String name = cNvPr.getStr("name");
                if (StrUtil.isNotEmpty(name)) {
                    String strImageEmbed = updateImageEmbed(pic);
                    if (strImageEmbed != null) {
                        mapConfig.put(strImageEmbed, name);
                    }
                }
            }
        }
    }

    /**
     * 获取嵌入式图片的 embed 信息。
     *
     * @param pic 图片的 JSONObject
     * @return embed 信息
     */
    private static String updateImageEmbed(JSONObject pic) {
        JSONObject blipFill = pic.getJSONObject("xdr:blipFill");
        if (blipFill != null) {
            JSONObject blip = blipFill.getJSONObject("a:blip");
            if (blip != null) {
                return blip.getStr("r:embed");
            }
        }
        return null;
    }

    /**
     * 处理 Zip 文件中的 relationship 条目，更新配置信息。
     *
     * @param zipInputStream Zip 输入流
     * @param mapConfig      配置信息的 map
     * @return 配置信息的 map
     * @throws IOException
     */
    private static Map<String, String> processCellImagesRels(ZipInputStream
                                                                     zipInputStream, Map<String, String> mapConfig) throws IOException {
        String content = IOUtils.toString(zipInputStream, StandardCharsets.UTF_8);
        JSONObject jsonObject = XML.toJSONObject(content);
        JSONObject relationships = jsonObject.getJSONObject("Relationships");
        if (relationships != null) {
            JSONArray relationshipArray = null;
            Object relationship = relationships.get("Relationship");

            if (relationship != null && relationship instanceof JSONArray) {
                relationshipArray = (JSONArray) relationship;
            } else if (relationship != null && relationship instanceof JSONObject) {
                relationshipArray = new JSONArray();
                relationshipArray.add(relationship);
            }
            if (relationshipArray != null) {
                return processRelationships(relationshipArray, mapConfig);
            }
        }
        return null;
    }

    /**
     * 处理 relationshipArray 中的关系项，更新配置信息。
     *
     * @param relationshipArray 关系项的 JSONArray
     * @param mapConfig         配置信息的 map
     * @return 配置信息的 map
     */
    private static Map<String, String> processRelationships(JSONArray
                                                                    relationshipArray, Map<String, String> mapConfig) {
        Map<String, String> mapRelationships = new HashedMap<>();
        for (int i = 0; i < relationshipArray.size(); i++) {
            JSONObject relaItem = relationshipArray.getJSONObject(i);
            if (relaItem != null) {
                String id = relaItem.getStr("Id");
                String value = "/xl/" + relaItem.getStr("Target");
                if (mapConfig.containsKey(id)) {
                    String strImageId = mapConfig.get(id);
                    mapRelationships.put(value, strImageId);
                }
            }
        }
        return mapRelationships;
    }

    /**
     * @param file 数据文件
     * @return {@link byte[]}
     * @description
     * @author bianhl
     * @date 2024/4/26 13:52
     */
    private byte[] getFileStream(File file) {
        try (InputStream inputStream = new FileInputStream(file)) {
            // 创建 ByteArrayOutputStream 来暂存流数据
            ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
            // 将 inputStream 读取到 byteArrayOutputStream 中
            byte[] buffer = new byte[1024];
            int length;
            while ((length = inputStream.read(buffer)) != -1) {
                byteArrayOutputStream.write(buffer, 0, length);
            }
            // 将 byteArrayOutputStream 的内容获取为字节数组
            return byteArrayOutputStream.toByteArray();
        } catch (IOException e) {
            return null;
        }
    }
}
```
2. 核心辅助工具类
```java
import lombok.extern.slf4j.Slf4j;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.reflection.ReflectionException;

import java.io.Serializable;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.reflect.*;
import java.security.AccessController;
import java.security.PrivilegedAction;
import java.util.Locale;
import java.util.function.Function;

/**
 * Lambda 解析工具类 (整体代码源自 mybatis-plus中的相关的工具类)
 *
 * @author HCL, MieMie
 * @since 2018-05-10
 */
public final class LambdaUtils {

    @FunctionalInterface
    public interface SFunction<T, R> extends Function<T, R>, Serializable {
    }

    public interface StringPool {

        String AMPERSAND = "&";
        String AND = "and";
        String AT = "@";
        String ASTERISK = "*";
        String STAR = ASTERISK;
        String BACK_SLASH = "\\";
        String COLON = ":";
        String COMMA = ",";
        String DASH = "-";
        String DOLLAR = "$";
        String DOT = ".";
        String DOTDOT = "..";
        String DOT_CLASS = ".class";
        String DOT_JAVA = ".java";
        String DOT_XML = ".xml";
        String EMPTY = "";
        String EQUALS = "=";
        String FALSE = "false";
        String SLASH = "/";
        String HASH = "#";
        String HAT = "^";
        String LEFT_BRACE = "{";
        String LEFT_BRACKET = "(";
        String LEFT_CHEV = "<";
        String DOT_NEWLINE = ",\n";
        String NEWLINE = "\n";
        String N = "n";
        String NO = "no";
        String NULL = "null";
        String NUM = "NUM";
        String OFF = "off";
        String ON = "on";
        String PERCENT = "%";
        String PIPE = "|";
        String PLUS = "+";
        String QUESTION_MARK = "?";
        String EXCLAMATION_MARK = "!";
        String QUOTE = "\"";
        String RETURN = "\r";
        String TAB = "\t";
        String RIGHT_BRACE = "}";
        String RIGHT_BRACKET = ")";
        String RIGHT_CHEV = ">";
        String SEMICOLON = ";";
        String SINGLE_QUOTE = "'";
        String BACKTICK = "`";
        String SPACE = " ";
        String SQL = "sql";
        String TILDA = "~";
        String LEFT_SQ_BRACKET = "[";
        String RIGHT_SQ_BRACKET = "]";
        String TRUE = "true";
        String UNDERSCORE = "_";
        String UTF_8 = "UTF-8";
        String US_ASCII = "US-ASCII";
        String ISO_8859_1 = "ISO-8859-1";
        String Y = "y";
        String YES = "yes";
        String ONE = "1";
        String ZERO = "0";
        String DOLLAR_LEFT_BRACE = "${";
        String HASH_LEFT_BRACE = "#{";
        String CRLF = "\r\n";

        String HTML_NBSP = "&nbsp;";
        String HTML_AMP = "&amp";
        String HTML_QUOTE = "&quot;";
        String HTML_LT = "&lt;";
        String HTML_GT = "&gt;";

        // ---------------------------------------------------------------- array

        String[] EMPTY_ARRAY = new String[0];

        byte[] BYTES_NEW_LINE = StringPool.NEWLINE.getBytes();
    }

    /**
     * Lambda 信息
     * <p>
     * Created by hcl at 2021/5/14
     */
    public interface LambdaMeta {

        /**
         * 获取 lambda 表达式实现方法的名称
         *
         * @return lambda 表达式对应的实现方法名称
         */
        String getImplMethodName();

        /**
         * 实例化该方法的类
         *
         * @return 返回对应的类名称
         */
        Class<?> getInstantiatedClass();

    }

    public static class ShadowLambdaMeta implements LambdaMeta {
        private final SerializedLambda lambda;

        public ShadowLambdaMeta(SerializedLambda lambda) {
            this.lambda = lambda;
        }

        @Override
        public String getImplMethodName() {
            return lambda.getImplMethodName();
        }

        @Override
        public Class<?> getInstantiatedClass() {
            String instantiatedMethodType = lambda.getInstantiatedMethodType();
            String instantiatedType = instantiatedMethodType.substring(2, instantiatedMethodType.indexOf(StringPool.SEMICOLON)).replace(StringPool.SLASH, StringPool.DOT);
            return toClassConfident(instantiatedType, lambda.getCapturingClass().getClassLoader());
        }
    }

    /**
     * 在 IDEA 的 Evaluate 中执行的 Lambda 表达式元数据需要使用该类处理元数据
     * <p>
     * Create by hcl at 2021/5/17
     */
    public static class IdeaProxyLambdaMeta implements LambdaMeta {
        private final Class<?> clazz;
        private final String name;

        public IdeaProxyLambdaMeta(Proxy func) {
            InvocationHandler handler = Proxy.getInvocationHandler(func);
            try {
                MethodHandle dmh = (MethodHandle) AccessController.doPrivileged(new SetAccessibleAction<>(handler.getClass().getDeclaredField("val$target"))).get(handler);
                Executable executable = MethodHandles.reflectAs(Executable.class, dmh);
                clazz = executable.getDeclaringClass();
                name = executable.getName();
            } catch (IllegalAccessException | NoSuchFieldException e) {
                throw new IllegalStateException(e);
            }
        }

        @Override
        public String getImplMethodName() {
            return name;
        }

        @Override
        public Class<?> getInstantiatedClass() {
            return clazz;
        }

        @Override
        public String toString() {
            return clazz.getSimpleName() + "::" + name;
        }

    }

    public static class SetAccessibleAction<T extends AccessibleObject> implements PrivilegedAction<T> {
        private final T obj;

        public SetAccessibleAction(T obj) {
            this.obj = obj;
        }

        @Override
        public T run() {
            obj.setAccessible(true);
            return obj;
        }

    }

    /**
     * Created by hcl at 2021/5/14
     */
    @Slf4j
    public static class ReflectLambdaMeta implements LambdaMeta {
        private static final Field FIELD_CAPTURING_CLASS;

        static {
            Field fieldCapturingClass;
            try {
                Class<java.lang.invoke.SerializedLambda> aClass = java.lang.invoke.SerializedLambda.class;
                fieldCapturingClass = (Field) AccessController.doPrivileged(new SetAccessibleAction(aClass.getDeclaredField("capturingClass")));
            } catch (Throwable e) {
                // 解决高版本 jdk 的问题 gitee: https://gitee.com/baomidou/mybatis-plus/issues/I4A7I5
                log.warn(e.getMessage());
                fieldCapturingClass = null;
            }
            FIELD_CAPTURING_CLASS = fieldCapturingClass;
        }

        private final java.lang.invoke.SerializedLambda lambda;

        public ReflectLambdaMeta(java.lang.invoke.SerializedLambda lambda) {
            this.lambda = lambda;
        }

        @Override
        public String getImplMethodName() {
            return lambda.getImplMethodName();
        }

        @Override
        public Class<?> getInstantiatedClass() {
            String instantiatedMethodType = lambda.getInstantiatedMethodType();
            String instantiatedType = instantiatedMethodType.substring(2, instantiatedMethodType.indexOf(StringPool.SEMICOLON)).replace(StringPool.SLASH, StringPool.DOT);
            return toClassConfident(instantiatedType, getCapturingClassClassLoader());
        }

        private ClassLoader getCapturingClassClassLoader() {
            // 如果反射失败，使用默认的 classloader
            if (FIELD_CAPTURING_CLASS == null) {
                return null;
            }
            try {
                return ((Class<?>) FIELD_CAPTURING_CLASS.get(lambda)).getClassLoader();
            } catch (IllegalAccessException e) {
                throw new IllegalStateException(e);
            }
        }

    }


    /**
     * 该缓存可能会在任意不定的时间被清除
     *
     * @param func 需要解析的 lambda 对象
     * @param <T>  类型，被调用的 Function 对象的目标类型
     * @return 返回解析后的结果
     */
    public static <T> LambdaMeta extract(SFunction<T, ?> func) {
        // 1. IDEA 调试模式下 lambda 表达式是一个代理
        if (func instanceof Proxy) {
            return new IdeaProxyLambdaMeta((Proxy) func);
        }
        // 2. 反射读取
        try {
            Method method = func.getClass().getDeclaredMethod("writeReplace");
            return new ReflectLambdaMeta((java.lang.invoke.SerializedLambda) AccessController.doPrivileged(new SetAccessibleAction<>(method)).invoke(func));
        } catch (Throwable e) {
            // 3. 反射失败使用序列化的方式读取
            return new ShadowLambdaMeta(SerializedLambda.extract(func));
        }
    }


    /* ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓  反射工具 ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓ */
    public static String methodToProperty(String name) {
        if (name.startsWith("is")) {
            name = name.substring(2);
        } else if (name.startsWith("get") || name.startsWith("set")) {
            name = name.substring(3);
        } else {
            throw new ReflectionException("Error parsing property name '" + name + "'.  Didn't start with 'is', 'get' or 'set'.");
        }

        if (name.length() == 1 || (name.length() > 1 && !Character.isUpperCase(name.charAt(1)))) {
            name = name.substring(0, 1).toLowerCase(Locale.ENGLISH) + name.substring(1);
        }

        return name;
    }

    public static boolean isProperty(String name) {
        return isGetter(name) || isSetter(name);
    }

    public static boolean isGetter(String name) {
        return (name.startsWith("get") && name.length() > 3) || (name.startsWith("is") && name.length() > 2);
    }

    public static boolean isSetter(String name) {
        return name.startsWith("set") && name.length() > 3;
    }

    private static ClassLoader systemClassLoader;

    static {
        try {
            systemClassLoader = ClassLoader.getSystemClassLoader();
        } catch (SecurityException ignored) {
            // AccessControlException on Google App Engine
        }
    }

    /**
     * <p>
     * 请仅在确定类存在的情况下调用该方法
     * </p>
     *
     * @param name 类名称
     * @return 返回转换后的 Class
     */
    public static Class<?> toClassConfident(String name) {
        return toClassConfident(name, null);
    }

    /**
     * @param name
     * @param classLoader
     * @return
     * @since 3.4.3
     */
    public static Class<?> toClassConfident(String name, ClassLoader classLoader) {
        try {
            return loadClass(name, getClassLoaders(classLoader));
        } catch (ClassNotFoundException e) {
            throw new IllegalArgumentException("找不到指定的class！请仅在明确确定会有 class 的时候，调用该方法", e);
        }
    }

    private static ClassLoader[] getClassLoaders(ClassLoader classLoader) {
        return new ClassLoader[]{
                classLoader,
                Resources.getDefaultClassLoader(),
                Thread.currentThread().getContextClassLoader(),
                LambdaUtils.class.getClassLoader(),
                systemClassLoader};
    }

    private static Class<?> loadClass(String className, ClassLoader[] classLoaders) throws ClassNotFoundException {
        for (ClassLoader classLoader : classLoaders) {
            if (classLoader != null) {
                try {
                    return Class.forName(className, true, classLoader);
                } catch (ClassNotFoundException e) {
                    // ignore
                }
            }
        }
        throw new ClassNotFoundException("Cannot find class: " + className);
    }
    /* ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑  反射工具 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ */

}

```
3. 辅助类（不可作为静态内部类，必须做独立类）
```java

import java.io.*;

/**
 * 当前类是 {@link java.lang.invoke.SerializedLambda } 的一个镜像
 * 此类不能被定义为静态内部类,不然在反序列化当前对象的时候,就会存在问题;
 * <p>
 * Create by hcl at 2020/7/17
 */
@SuppressWarnings("ALL")
public class SerializedLambda implements Serializable {
    private static final long serialVersionUID = 8025925345765570181L;

    private Class<?> capturingClass;
    private String functionalInterfaceClass;
    private String functionalInterfaceMethodName;
    private String functionalInterfaceMethodSignature;
    private String implClass;
    private String implMethodName;
    private String implMethodSignature;
    private int implMethodKind;
    private String instantiatedMethodType;
    private Object[] capturedArgs;

    public static SerializedLambda extract(Serializable serializable) {
        try (ByteArrayOutputStream baos = new ByteArrayOutputStream();
             ObjectOutputStream oos = new ObjectOutputStream(baos)) {
            oos.writeObject(serializable);
            oos.flush();
            try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(baos.toByteArray())) {
                @Override
                protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
                    Class<?> clazz = super.resolveClass(desc);
                    return clazz == java.lang.invoke.SerializedLambda.class ? SerializedLambda.class : clazz;
                }

            }) {
                return (SerializedLambda) ois.readObject();
            }
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalStateException(e);
        }
    }

    public String getInstantiatedMethodType() {
        return instantiatedMethodType;
    }

    public Class<?> getCapturingClass() {
        return capturingClass;
    }

    public String getImplMethodName() {
        return implMethodName;
    }
}

```

##### 二、使用
1. 在待导入的模板类中增加注解
```java

import cn.afterturn.easypoi.excel.annotation.Excel;
public class PitfallReportImport{
    //此处使用的是easypoi中的@Excel注解，此块主要是为了拿到字段对应到Excel文件中的表头的汉字名称，如果不用这个注解也可以，只需要更改工具类中对应的方法即可
    @Excel(name = "隐患照片")
    private String checkImg;//当前可以定义为String类型或者File类型，会根据类型进行填充；
}

```
2. 工具类使用
```java

List<PitfallReportImport> dicImportVos = ExcelUtil.importExcel(file, 1, 1, PitfallReportImport.class);//此处使用easypoi填充List<PitfallReportImport>集合数据
ExcelPicUtilPro.flatExcelImg2List(file,  1,  dicImportVos, PitfallReportImport::getCheckImg);
//调用完工具类后，dicImportVos中条目的checkImg会被填充上对应的图片路径（前提是图片存在）

```
