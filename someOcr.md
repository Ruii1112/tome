### シーケンス
```mermaid
sequenceDiagram
    participant Frontend as Frontend (React)
    participant Backend as Backend (Seasar2 API)
    participant Thread1 as Thread #1
    participant Thread2 as Thread #2
    participant Thread3 as Thread #3
    participant OCR as OCR API (AWS)
    participant DB as Database

    Frontend->>Backend: POST /api/ocr/upload (PDFファイル3枚をアップロード)
    Backend->>Backend: uploadId を生成（例: xyz123）

    Backend->>Thread1: スレッドで file1.pdf のOCR処理を開始
    Backend->>Thread2: スレッドで file2.pdf のOCR処理を開始
    Backend->>Thread3: スレッドで file3.pdf のOCR処理を開始

    Backend-->>Frontend: { uploadId: "xyz123" } を返却

    Note over Backend,Thread3: ExecutorService によりスレッドを並列起動

    Thread1->>OCR: POST /ocr (file1.pdf)
    Thread2->>OCR: POST /ocr (file2.pdf)
    Thread3->>OCR: POST /ocr (file3.pdf)

    OCR-->>Thread1: OCR結果 (text1)
    OCR-->>Thread2: OCR結果 (text2)
    OCR-->>Thread3: OCR結果 (text3)

    Thread1->>DB: INSERT INTO ocr_result (file1, status=READY)
    Thread2->>DB: INSERT INTO ocr_result (file2, status=READY)
    Thread3->>DB: INSERT INTO ocr_result (file3, status=READY)

    loop 数秒ごとに結果確認（ポーリング）
        Frontend->>Backend: GET /api/ocr/results?uploadId=xyz123
        Backend->>DB: ocr_result テーブルから該当ファイルを検索
        DB-->>Backend: 検索結果（各ファイルのstatus, text）
        Backend-->>Frontend: JSONで結果を返す
    end

    alt 1件取得リクエスト
        Frontend->>Backend: GET /api/ocr/result (id=123 フォームで送信)
        Backend->>DB: ocr_result テーブルから id=123 を検索
        DB-->>Backend: レコード詳細 (fileName, status, text)
        Backend-->>Frontend: JSONで1件のOCR結果を返却
    end
```

### Action

```Java
package your.package.web.api;

import java.util.List;
import javax.annotation.Resource;
import javax.servlet.http.HttpServletRequest;

import org.seasar.struts.annotation.Execute;
import org.seasar.framework.beans.util.Beans;
import org.apache.commons.fileupload.FileItem;
import org.apache.struts.upload.FormFile;

import your.package.dto.OcrResultDto;
import your.package.service.OcrService;
import your.package.form.OcrUploadForm;

public class OcrAction {

    @Resource
    private OcrService ocrService;

    public OcrUploadForm ocrUploadForm;

    @Execute(validator = false)
    public String upload() {
        String uploadId = UUID.randomUUID().toString(); // リクエストをグルーピングするための一意のid
        List<Integer> recordIds = ocrService.handleUpload(uploadId, ocrUploadForm.files);

        Map<String, Object> response = new HashMap<>();
        response.put("uploadId", uploadId);
        response.put("recordIds", recordIds);
        return toJson(response);
    }

    @Execute(validator = false)
    public String results() {
        String uploadId = ocrUploadForm.uploadId;
        List<OcrResultDto> results = ocrService.getResults(uploadId);
        return toJson(results);  // Jacksonなどを使ってJSONに変換
    }

    @Execute(validator = false)
    public String result() {
        String idStr = ocrResultForm.id;
        if (idStr == null || idStr.isEmpty()) {
            リクエストで何もなかったら、400返す
            return toJson(Collections.singletonMap("error", "id is required"));
        }

        try {
            int id = Integer.parseInt(idStr);
            OcrResultDto dto = ocrService.getResult(id);
            if (dto == null) {
                return toJson(Collections.singletonMap("error", "Not found"));
            }
            return toJson(dto);
        } catch (NumberFormatException e) {
            return toJson(Collections.singletonMap("error", "Invalid id format"));
        }
    }


    private String toJson(Object obj) {
        try {
            return new com.fasterxml.jackson.databind.ObjectMapper().writeValueAsString(obj);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

### Form

```Java
package your.package.form;

import java.util.List;
import org.apache.struts.upload.FormFile;

public class OcrUploadForm {
    public List<FormFile> files;
    public String id;
    public String uploadId;
}
```

### Service

```Java
package your.package.service;

import java.util.*;
import java.util.concurrent.*;
import java.util.stream.Collectors;

import javax.annotation.Resource;
import org.apache.commons.io.IOUtils;

import your.package.dao.OcrResultDao;
import your.package.dto.OcrResultDto;
import your.package.entity.OcrResult;

public class OcrService {

    @Resource
    public OcrResultDao ocrResultDao;

    private ExecutorService executor = Executors.newFixedThreadPool(3); // スレッド数を3に

    public List<Integer> handleUpload(String uploadId, List<FormFile> files) {
        List<Integer> recordIds = new ArrayList<>();

        for (FormFile file : files) {
            // まずDBに空のレコードをinsertしてidを取得
            OcrResult result = new OcrResult();
            result.uploadId = uploadId;
            result.fileName = file.getFileName();
            result.status = "PROCESSING";
            result.createdAt = new Date();

            int insertedId = ocrResultDao.insert(result);  // insertでDBの主キーIDが返る想定
            recordIds.add(insertedId);

            byte[] fileBytes = IOUtils.toByteArray(file.getInputStream());
            // 非同期処理
            executor.submit(() -> {
                try {
                    String text = callOcrApi(fileBytes);

                    // update処理（insert時のIDでレコード特定）
                    OcrResult updateResult = new OcrResult();
                    updateResult.id = insertedId;
                    updateResult.text = text;
                    updateResult.status = "READY";
                    ocrResultDao.update(updateResult);

                } catch (Exception e) {
                    OcrResult failResult = new OcrResult();
                    failResult.id = insertedId;
                    failResult.text = "";
                    failResult.status = "FAILED";
                    ocrResultDao.update(failResult);
                }
            });
        }
        return recordIds;
    }


    public List<OcrResultDto> getResults(String uploadId) {
        List<OcrResult> entities = ocrResultDao.findByUploadId(uploadId);
        return entities.stream().map(e -> {
            OcrResultDto dto = new OcrResultDto();
            dto.fileName = e.fileName;
            dto.status = e.status;
            dto.text = e.text;
            return dto;
        }).collect(Collectors.toList());
    }

    public OcrResultDto getResult(int id) {
        OcrResult entity = ocrResultDao.findById(id);
        if (entity == null) return null;

        OcrResultDto dto = new OcrResultDto();
        dto.fileName = entity.fileName;
        dto.status = entity.status;
        dto.text = entity.text;
        return dto;
    }


    private String callOcrApi(byte[] fileBytes) {
        // OCR API へのHTTPリクエスト処理（例: OkHttp や Apache HttpClient）
        // ダミー実装
        return "OCR結果のダミー";
    }
}
```

### dto

```Java
package your.package.dto;

public class OcrResultDto {
    public String fileName;
    public String status;
    public String text;
}
```

### entity & dao

```Java
package your.package.entity;

import java.util.Date;

public class OcrResult {
    public Integer id;
    public String uploadId;
    public String fileName;
    public String text;
    public String status;
    public Date createdAt;
}
```

```Java
package your.package.dao;

import java.util.List;
import your.package.entity.OcrResult;

public interface OcrResultDao {
    List<OcrResult> findByUploadId(String uploadId);
    int insert(OcrResult entity);
    int update(OcrResult entity);
    OcrResult findById(int id);
}
```