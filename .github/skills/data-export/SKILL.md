---
name: data-export
description: >-
  Domain knowledge for exporting data to CSV and Excel formats in Java. Covers Apache POI
  for XLSX generation (XSSF/SXSSF for streaming), OpenCSV for CSV, and Spring Boot
  streaming response patterns. Use when building data export features.
---

# Data Export Skill

## Purpose

This skill provides patterns for exporting application data to CSV and Excel (XLSX) formats
in a Spring Boot application. It covers Apache POI for Excel generation, OpenCSV for CSV
writing, and streaming response patterns for large datasets that avoid loading everything
into memory.

## Maven Dependencies

```xml
<!-- Apache POI for Excel -->
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.3.0</version>
</dependency>

<!-- OpenCSV for CSV -->
<dependency>
    <groupId>com.opencsv</groupId>
    <artifactId>opencsv</artifactId>
    <version>5.9</version>
</dependency>
```

## Apache POI for Excel

### Basic XSSFWorkbook (Small Datasets)

```java
import org.apache.poi.xssf.usermodel.*;
import org.apache.poi.ss.usermodel.*;

public class ExcelExporter {

    public XSSFWorkbook exportToExcel(List<FieldRecord> fields) {
        XSSFWorkbook workbook = new XSSFWorkbook();
        XSSFSheet sheet = workbook.createSheet("API Fields");

        // Header style
        CellStyle headerStyle = workbook.createCellStyle();
        Font headerFont = workbook.createFont();
        headerFont.setBold(true);
        headerStyle.setFont(headerFont);
        headerStyle.setFillForegroundColor(IndexedColors.LIGHT_CORNFLOWER_BLUE.getIndex());
        headerStyle.setFillPattern(FillPatternType.SOLID_FOREGROUND);

        // Header row
        String[] headers = {"Field Name", "Type", "Format", "Description", "Required", "Schema"};
        Row headerRow = sheet.createRow(0);
        for (int i = 0; i < headers.length; i++) {
            Cell cell = headerRow.createCell(i);
            cell.setCellValue(headers[i]);
            cell.setCellStyle(headerStyle);
        }

        // Data rows
        int rowNum = 1;
        for (FieldRecord field : fields) {
            Row row = sheet.createRow(rowNum++);
            row.createCell(0).setCellValue(field.fieldName());
            row.createCell(1).setCellValue(field.fieldType());
            row.createCell(2).setCellValue(field.format() != null ? field.format() : "");
            row.createCell(3).setCellValue(field.description() != null ? field.description() : "");
            row.createCell(4).setCellValue(field.required() ? "Yes" : "No");
            row.createCell(5).setCellValue(field.schemaName());
        }

        // Auto-size columns
        for (int i = 0; i < headers.length; i++) {
            sheet.autoSizeColumn(i);
        }

        return workbook;
    }
}
```

### SXSSFWorkbook for Large Datasets (Streaming)

SXSSFWorkbook writes rows to disk in chunks, keeping only a sliding window of rows in memory.
Use this for datasets over ~10,000 rows.

```java
import org.apache.poi.xssf.streaming.SXSSFWorkbook;
import org.apache.poi.xssf.streaming.SXSSFSheet;

public class StreamingExcelExporter {

    // Window size: number of rows kept in memory before flushing to disk
    private static final int ROW_WINDOW_SIZE = 500;

    public void exportLargeDataset(OutputStream outputStream, DataProvider provider) throws IOException {
        try (SXSSFWorkbook workbook = new SXSSFWorkbook(ROW_WINDOW_SIZE)) {
            workbook.setCompressTempFiles(true);
            SXSSFSheet sheet = workbook.createSheet("Export");

            // Write header
            Row headerRow = sheet.createRow(0);
            String[] headers = provider.getHeaders();
            for (int i = 0; i < headers.length; i++) {
                headerRow.createCell(i).setCellValue(headers[i]);
            }

            // Stream data in pages
            int rowNum = 1;
            int page = 0;
            List<Object[]> batch;

            do {
                batch = provider.fetchPage(page++, 1000);
                for (Object[] record : batch) {
                    Row row = sheet.createRow(rowNum++);
                    for (int col = 0; col < record.length; col++) {
                        Cell cell = row.createCell(col);
                        setCellValue(cell, record[col]);
                    }
                }
            } while (!batch.isEmpty());

            workbook.write(outputStream);
        }
    }

    private void setCellValue(Cell cell, Object value) {
        if (value == null) {
            cell.setBlank();
        } else if (value instanceof Number n) {
            cell.setCellValue(n.doubleValue());
        } else if (value instanceof Boolean b) {
            cell.setCellValue(b);
        } else if (value instanceof java.time.LocalDate ld) {
            cell.setCellValue(ld.toString());
        } else {
            cell.setCellValue(value.toString());
        }
    }
}
```

## OpenCSV for CSV

### Basic CSVWriter

```java
import com.opencsv.CSVWriter;
import java.io.*;

public class CsvExporter {

    public void exportToCsv(Writer writer, List<FieldRecord> fields) throws IOException {
        try (CSVWriter csvWriter = new CSVWriter(writer)) {
            // Header
            csvWriter.writeNext(new String[]{
                "Field Name", "Type", "Format", "Description", "Required", "Schema"
            });

            // Data
            for (FieldRecord field : fields) {
                csvWriter.writeNext(new String[]{
                    field.fieldName(),
                    field.fieldType(),
                    field.format(),
                    field.description(),
                    field.required() ? "Yes" : "No",
                    field.schemaName()
                });
            }
        }
    }
}
```

### Bean-to-CSV with Custom Column Mapping

```java
import com.opencsv.bean.*;
import com.opencsv.bean.StatefulBeanToCsv;
import com.opencsv.bean.StatefulBeanToCsvBuilder;

// Annotate your class for column mapping
public class FieldCsvRow {

    @CsvBindByName(column = "Field Name")
    @CsvBindByPosition(position = 0)
    private String fieldName;

    @CsvBindByName(column = "Type")
    @CsvBindByPosition(position = 1)
    private String fieldType;

    @CsvBindByName(column = "Description")
    @CsvBindByPosition(position = 2)
    private String description;

    // constructors, getters omitted for brevity
}

// Export using bean-to-CSV
public void exportBeansCsv(Writer writer, List<FieldCsvRow> rows) throws Exception {
    StatefulBeanToCsv<FieldCsvRow> beanToCsv = new StatefulBeanToCsvBuilder<FieldCsvRow>(writer)
        .withQuotechar(CSVWriter.DEFAULT_QUOTE_CHARACTER)
        .withSeparator(CSVWriter.DEFAULT_SEPARATOR)
        .withOrderedResults(true)
        .build();

    beanToCsv.write(rows);
}
```

## Spring Boot Streaming Response

### StreamingResponseBody Pattern

```java
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.servlet.mvc.method.annotation.StreamingResponseBody;

@RestController
@RequestMapping("/api/v1/export")
public class ExportController {

    private final ExportService exportService;

    public ExportController(ExportService exportService) {
        this.exportService = exportService;
    }

    @GetMapping("/excel")
    public ResponseEntity<StreamingResponseBody> exportExcel(
            @RequestParam(required = false) String schemaName) {

        String filename = "api-fields-export.xlsx";

        StreamingResponseBody body = outputStream ->
            exportService.exportToExcel(outputStream, schemaName);

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .contentType(MediaType.parseMediaType(
                "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
            .body(body);
    }

    @GetMapping("/csv")
    public ResponseEntity<StreamingResponseBody> exportCsv(
            @RequestParam(required = false) String schemaName) {

        String filename = "api-fields-export.csv";

        StreamingResponseBody body = outputStream -> {
            try (OutputStreamWriter writer = new OutputStreamWriter(outputStream, StandardCharsets.UTF_8)) {
                // Write BOM for Excel compatibility
                outputStream.write(0xEF);
                outputStream.write(0xBB);
                outputStream.write(0xBF);
                exportService.exportToCsv(writer, schemaName);
            }
        };

        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
            .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
            .body(body);
    }
}
```

## Large Dataset Handling

### Paginated Export Service

```java
@Service
public class ExportService {

    private final FieldRepository fieldRepository;
    private static final int BATCH_SIZE = 1000;

    public ExportService(FieldRepository fieldRepository) {
        this.fieldRepository = fieldRepository;
    }

    public void exportToExcel(OutputStream outputStream, String schemaFilter) throws IOException {
        try (SXSSFWorkbook workbook = new SXSSFWorkbook(500)) {
            SXSSFSheet sheet = workbook.createSheet("Fields");
            writeExcelHeader(sheet);

            int rowNum = 1;
            int page = 0;
            Page<ApiField> batch;

            do {
                batch = fieldRepository.findBySchemaFilter(
                    schemaFilter, PageRequest.of(page++, BATCH_SIZE));

                for (ApiField field : batch.getContent()) {
                    Row row = sheet.createRow(rowNum++);
                    writeFieldToRow(row, field);
                }
            } while (batch.hasNext());

            workbook.write(outputStream);
        }
    }

    public void exportToCsv(Writer writer, String schemaFilter) throws IOException {
        try (CSVWriter csvWriter = new CSVWriter(writer)) {
            csvWriter.writeNext(new String[]{"Name", "Type", "Description", "Schema"});

            int page = 0;
            Page<ApiField> batch;

            do {
                batch = fieldRepository.findBySchemaFilter(
                    schemaFilter, PageRequest.of(page++, BATCH_SIZE));

                for (ApiField field : batch.getContent()) {
                    csvWriter.writeNext(fieldToCsvRow(field));
                }

                csvWriter.flush();
            } while (batch.hasNext());
        }
    }
}
```

### Memory Management Tips

- **SXSSFWorkbook**: Always use for Excel exports > 10k rows. Set `setCompressTempFiles(true)`.
- **CSVWriter**: Call `flush()` periodically during large exports.
- **StreamingResponseBody**: Spring runs this in a separate thread; configure `spring.mvc.async.request-timeout` appropriately.
- **Pagination**: Fetch data in batches (1000 rows is a good default) to avoid loading the entire result set.
- **Close resources**: Always use try-with-resources for workbooks and writers.
- **Temp file cleanup**: SXSSFWorkbook creates temp files; calling `dispose()` after `write()` cleans them up (handled by try-with-resources via `close()`).
