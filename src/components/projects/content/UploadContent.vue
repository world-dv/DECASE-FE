<template>
  <div class="upload-content">
    <div class="content-header">
      <h2 class="content-title">📄 업로드된 파일 미리보기</h2>
      <div class="file-info">
        <span class="file-id">문서 ID: {{ docId }}</span>
        <span v-if="previewData" class="file-name">{{
          previewData.fileName
        }}</span>
        <button
          @click="downloadFile"
          class="refresh-button"
          :disabled="loading"
        >
          {{ loading ? "🔄 다운로드중..." : "📥 다운로드" }}
        </button>
      </div>
    </div>

    <div
        v-if="previewData && previewData.docDescription"
        class="content-summary"
    >
      <h3 class="summary-title">📌 요약</h3>
      <div class="summary-body" v-html="previewData.docDescription"></div>
    </div>

    <div v-if="loading" class="loading-state">
      <div class="loading-spinner"></div>
      <p>파일을 불러오는 중...</p>
    </div>

    <div v-else-if="error" class="error-state">
      <div class="error-icon">❌</div>
      <h3>파일을 불러올 수 없습니다</h3>
      <p>{{ error }}</p>
      <button @click="refreshPreview" class="retry-button">다시 시도</button>
    </div>

    <div v-else-if="previewData" class="preview-container">
      <!-- PDF 미리보기 -->
      <div v-if="previewData.fileType === 'pdf'" class="pdf-preview">
        <iframe
          :src="previewData.previewUrl"
          class="pdf-iframe"
          title="PDF 미리보기"
        ></iframe>
      </div>

      <!-- DOCX 미리보기 -->
      <div v-else-if="previewData.fileType === 'docx'" class="docx-preview">
        <div class="document-content" v-html="previewData.htmlContent"></div>
      </div>

      <!-- Excel 미리보기 -->
      <div
        v-else-if="
          previewData.fileType === 'csv' ||
          previewData.fileType === 'xlsx' ||
          previewData.fileType === 'xls'
        "
        class="excel-preview"
      >
        <div
          class="excel-tabs"
          v-if="previewData.sheets && previewData.sheets.length > 1"
        >
          <button
            v-for="(sheet, index) in previewData.sheets"
            :key="index"
            @click="activeSheet = index"
            :class="['tab-button', { active: activeSheet === index }]"
          >
            {{ sheet.sheetName }}
          </button>
        </div>

        <div
          v-if="previewData.sheets && previewData.sheets.length > 0"
          class="excel-sheet"
        >
          <div class="sheet-info">
            <span>{{ previewData.sheets[activeSheet].sheetName }}</span>
            <span class="sheet-size">
              ({{ previewData.sheets[activeSheet].totalRows }}행 ×
              {{ previewData.sheets[activeSheet].totalCols }}열)
            </span>
          </div>

          <div class="excel-table-container">
            <table class="excel-table">
              <tbody>
                <tr
                  v-for="(row, rowIndex) in previewData.sheets[activeSheet]
                    .data"
                  :key="rowIndex"
                >
                  <td
                    v-for="(cell, cellIndex) in row"
                    :key="cellIndex"
                    class="excel-cell"
                  >
                    {{ cell }}
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <div
            class="preview-notice"
            v-if="previewData.sheets[activeSheet].totalRows > 100"
          >
            * 처음 100행만 미리보기로 표시됩니다.
          </div>
        </div>
      </div>

      <!-- 기타 파일 타입 -->
      <div v-else class="unsupported-preview">
        <div class="unsupported-icon">📎</div>
        <h3>미리보기를 지원하지 않는 파일 형식입니다</h3>
        <p>파일 타입: {{ previewData.fileType }}</p>
        <button @click="downloadFile" class="download-button">
          📥 파일 다운로드
        </button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, watch } from "vue";

const props = defineProps({
  docId: {
    type: String,
    required: true,
  },
  file: {
    type: Object,
    required: false,
  },
});

const loading = ref(false);
const error = ref(null);
const previewData = ref(null);
const activeSheet = ref(0);

// 파일 확장자 추출 함수
const getFileExtension = (fileName) => {
  if (!fileName) return "";
  return fileName.split(".").pop().toLowerCase();
};

// API에서 파일 미리보기 데이터 로드
const loadPreview = async () => {
  if (!props.docId) return;

  loading.value = true;
  error.value = null;
  previewData.value = null;
  activeSheet.value = 0;

  try {
    console.log("파일 미리보기 로드:", props.docId);
    console.log("props.file:", props.file); // 파일 객체 확인

    // props에서 파일 정보가 있다면 사용, 없다면 docId로만 처리
    let fileType = "";
    let fileName = "";

    if (props.file && props.file.name) {
      fileName = props.file.name;
      fileType = getFileExtension(fileName);
      console.log("파일명:", fileName, "파일 타입:", fileType);
    }

    if (fileType === "pdf") {
      // PDF의 경우 preview 엔드포인트만 사용
      previewData.value = {
        fileType: "pdf",
        fileName: fileName,
        previewUrl: `/api/v1/documents/${props.docId}/preview`,
      };
    } else {
      // CSV, DOCX, XLSX 등은 info 엔드포인트 사용
      const infoResponse = await fetch(
        `/api/v1/documents/${props.docId}/info`,
        {
          method: "GET",
          headers: {
            "Content-Type": "application/json",
          },
        }
      );

      if (!infoResponse.ok) {
        throw new Error(`HTTP error! status: ${infoResponse.status}`);
      }

      const data = await infoResponse.json();
      console.log("미리보기 데이터:", data);
      previewData.value = data;
    }
  } catch (err) {
    console.error("미리보기 로드 오류:", err);
    error.value = err.message || "파일을 불러오는 중 오류가 발생했습니다.";
  } finally {
    loading.value = false;
  }
};
// 미리보기 새로고침
const refreshPreview = () => {
  loadPreview();
};

// 파일 다운로드
const downloadFile = async () => {
  if (!props.docId) {
    alert("문서 ID가 없습니다.");
    return;
  }

  loading.value = true;
  error.value = null;

  try {
    const downloadUrl = `/api/v1/documents/${props.docId}/downloads`;
    console.log("다운로드 URL:", downloadUrl);

    const response = await fetch(downloadUrl);
    if (!response.ok) throw new Error("다운로드 실패");

    const blob = await response.blob();
    const url = window.URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = props.file?.name || `document_${props.docId}`;
    link.style.display = "none";

    document.body.appendChild(link);
    link.click();

    setTimeout(() => {
      if (link.parentNode) {
        document.body.removeChild(link);
      }
      window.URL.revokeObjectURL(url);
    }, 100);
  } catch (error) {
    console.error("파일 다운로드 오류:", error);
    alert("파일 다운로드에 실패했습니다.");
  } finally {
    loading.value = false;
  }
};

// docId 변경 감지
watch(
  () => props.docId,
  (newDocId) => {
    if (newDocId) {
      loadPreview();
    }
  },
  { immediate: true }
);

onMounted(() => {
  if (props.docId) {
    loadPreview();
  }
});
</script>

<style scoped>
.upload-content {
  padding: 20px;
  height: calc(100vh - 64px);
  overflow-y: auto;
  background-color: #f8f9fa;
  font-family: "Pretendard", -apple-system, BlinkMacSystemFont, "Segoe UI",
    Roboto, sans-serif;
}

.content-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  padding: 20px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.content-title {
  font-size: 24px;
  font-weight: 600;
  color: #1f2937;
  margin: 0;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 16px;
  flex-wrap: wrap;
}

.file-id {
  font-size: 14px;
  color: #6b7280;
  background: #f3f4f6;
  padding: 6px 12px;
  border-radius: 6px;
}

.file-name {
  font-size: 14px;
  color: #374151;
  background: #e5e7eb;
  padding: 6px 12px;
  border-radius: 6px;
  max-width: 300px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.refresh-button {
  padding: 8px 16px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s;
}

.refresh-button:hover:not(:disabled) {
  background: #2563eb;
}

.refresh-button:disabled {
  background: #9ca3af;
  cursor: not-allowed;
}

.loading-state,
.error-state,
.unsupported-preview {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 400px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e5e7eb;
  border-top: 4px solid #3b82f6;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 16px;
}

.error-icon,
.unsupported-icon {
  font-size: 48px;
  margin-bottom: 16px;
}

.retry-button,
.download-button {
  padding: 10px 20px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  cursor: pointer;
  margin-top: 16px;
  transition: background-color 0.2s;
}

.retry-button:hover,
.download-button:hover {
  background: #2563eb;
}

.preview-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  overflow: hidden;
}

.pdf-iframe {
  width: 100%;
  height: calc(100vh - 200px);
  border: none;
}

.docx-preview {
  padding: 40px;
  max-height: calc(100vh - 200px);
  overflow-y: auto;
}

.document-content {
  line-height: 1.6;
  color: #374151;
}

.document-content h1,
.document-content h2,
.document-content h3 {
  color: #1f2937;
  margin-top: 24px;
  margin-bottom: 12px;
}

.document-content p {
  margin-bottom: 12px;
}

.document-content table {
  width: 100%;
  border-collapse: collapse;
  margin: 16px 0;
}

.document-content th,
.document-content td {
  border: 1px solid #d1d5db;
  padding: 8px 12px;
  text-align: left;
}

.document-content th {
  background: #f9fafb;
  font-weight: 600;
}

/* Excel 미리보기 스타일 */
.excel-preview {
  display: flex;
  flex-direction: column;
  height: calc(100vh - 200px);
}

.excel-tabs {
  display: flex;
  border-bottom: 2px solid #e5e7eb;
  background: #f9fafb;
  padding: 0 20px;
  gap: 4px;
}

.tab-button {
  padding: 12px 20px;
  background: transparent;
  border: none;
  border-bottom: 3px solid transparent;
  font-size: 14px;
  font-weight: 500;
  color: #6b7280;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}

.tab-button:hover {
  color: #374151;
  background: #f3f4f6;
}

.tab-button.active {
  color: #3b82f6;
  border-bottom-color: #3b82f6;
  background: white;
}

.excel-sheet {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.sheet-info {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px 20px;
  background: #f8f9fa;
  border-bottom: 1px solid #e5e7eb;
  font-size: 14px;
  font-weight: 600;
  color: #374151;
}

.sheet-size {
  font-weight: 400;
  color: #6b7280;
}

.excel-table-container {
  flex: 1;
  overflow: auto;
  padding: 20px;
}

.excel-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
  min-width: 600px;
}

.excel-cell {
  border: 1px solid #d1d5db;
  padding: 8px 12px;
  background: white;
  vertical-align: top;
  max-width: 200px;
  word-wrap: break-word;
  white-space: pre-wrap;
}

.excel-cell:empty::before {
  content: "\00a0";
  color: transparent;
}

.excel-table tr:first-child .excel-cell {
  background: #f8f9fa;
  font-weight: 600;
  color: #374151;
}

.excel-table tr:hover .excel-cell {
  background: #f0f9ff;
}

.preview-notice {
  padding: 12px 20px;
  background: #fef3c7;
  color: #92400e;
  font-size: 14px;
  text-align: center;
  border-top: 1px solid #e5e7eb;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

/* 반응형 디자인 */
@media (max-width: 768px) {
  .upload-content {
    padding: 16px;
  }

  .content-header {
    flex-direction: column;
    gap: 16px;
    align-items: flex-start;
  }

  .file-info {
    flex-direction: column;
    gap: 12px;
    align-items: flex-start;
    width: 100%;
  }

  .file-name {
    max-width: 100%;
  }

  .content-title {
    font-size: 20px;
  }

  .pdf-iframe {
    height: calc(100vh - 300px);
  }

  .docx-preview {
    padding: 20px;
    max-height: calc(100vh - 300px);
  }

  .excel-tabs {
    padding: 0 16px;
    overflow-x: auto;
  }

  .tab-button {
    padding: 10px 16px;
    font-size: 13px;
    flex-shrink: 0;
  }

  .sheet-info {
    padding: 12px 16px;
    font-size: 13px;
  }

  .excel-table-container {
    padding: 16px;
  }

  .excel-cell {
    padding: 6px 8px;
    font-size: 12px;
    max-width: 120px;
  }
}


.content-summary {
  background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
  border: 1px solid #cbd5e1;
  border-radius: 12px;
  padding: 24px;
  margin: 16px 0;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
}

.summary-title {
  color: #1e293b;
  font-size: 1.1em;
  font-weight: 600;
  margin: 0 0 16px 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.summary-body {
  color: #475569;
  line-height: 1.6;
  font-size: 0.95em;
}

/* 테이블 스타일링 */
.summary-body :deep(.summary-table) {
  width: 100%;
  border-collapse: collapse;
  background: #ffffff;
  border-radius: 8px;
  overflow: hidden;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.summary-body :deep(.summary-table caption) {
  font-size: 1.1em;
  font-weight: 600;
  color: #1e293b;
  padding: 12px;
  background: linear-gradient(135deg, #ddd6fe 0%, #c7d2fe 100%);
  text-align: center;
}

.summary-body :deep(.summary-table th) {
  background: #f1f5f9;
  color: #334155;
  font-weight: 600;
  padding: 12px 16px;
  text-align: left;
  border-bottom: 2px solid #e2e8f0;
}

.summary-body :deep(.summary-table td) {
  padding: 12px 16px;
  border-bottom: 1px solid #f1f5f9;
  vertical-align: top;
}

.summary-body :deep(.summary-table td.category) {
  background: #f8fafc;
  font-weight: 600;
  color: #475569;
  min-width: 120px;
  border-right: 1px solid #e2e8f0;
}

/* 카테고리별 색상 구분 */
.summary-body :deep(.summary-table tr:has(td:contains("회의 핵심")) td.category) {
  background: linear-gradient(135deg, #dbeafe 0%, #bfdbfe 100%);
  color: #1e40af;
}

.summary-body :deep(.summary-table tr:has(td:contains("추가된")) td.category) {
  background: linear-gradient(135deg, #dcfce7 0%, #bbf7d0 100%);
  color: #166534;
}

.summary-body :deep(.summary-table tr:has(td:contains("수정된")) td.category) {
  background: linear-gradient(135deg, #fef3c7 0%, #fde68a 100%);
  color: #92400e;
}

.summary-body :deep(.summary-table tr:has(td:contains("삭제된")) td.category) {
  background: linear-gradient(135deg, #fee2e2 0%, #fecaca 100%);
  color: #991b1b;
}

/* 테이블 내 리스트 스타일링 */
.summary-body :deep(.summary-table ul) {
  margin: 0;
  padding-left: 16px;
}

.summary-body :deep(.summary-table li) {
  margin: 4px 0;
  line-height: 1.5;
}

/* 테이블 행 호버 효과 */
.summary-body :deep(.summary-table tbody tr:hover) {
  background: rgba(59, 130, 246, 0.05);
}

/* 반응형 테이블 */
@media (max-width: 768px) {
  .summary-body :deep(.summary-table) {
    font-size: 0.9em;
  }

  .summary-body :deep(.summary-table th),
  .summary-body :deep(.summary-table td) {
    padding: 8px 12px;
  }
}
</style>
