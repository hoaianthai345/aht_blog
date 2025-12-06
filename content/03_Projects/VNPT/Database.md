# Thiết kế database
## 1. PRINCIPLES (Định hướng thiết kế)

1. **Meeting-centric**: Mọi entity đều quy chiếu về cuộc họp.
    
2. **Traceability-by-design**: Tất cả action/decision/risk phải truy lại được “được phát biểu ở phút nào, bởi ai, trong cuộc họp nào”.
    
3. **Audit compliance**: Bắt buộc lưu:
    
    - Lịch sử sync sang Planner/Jira
        
    - Lịch sử edit minutes, confirm items
        
4. **Scalable AI compute**: Dữ liệu transcript được chia nhỏ dạng **TranscriptChunk** để phục vụ ASR + diarization + RAG timeline.
    
5. **RAG-ready**: Lưu metadata tài liệu + embedding vector store.

# SQLAlchemy Models
**SQLAlchemy Models** (còn gọi là **ORM Models**) là các lớp Python đại diện cho các bảng trong cơ sở dữ liệu.  
SQLAlchemy dùng các model này để:

- Tự động tạo bảng (hoặc migrate bảng)
    
- Map dữ liệu giữa Python object ⇄ Database row
    
- Cho phép bạn thao tác với database bằng Python thay vì viết SQL thủ công
    

Nói ngắn gọn:  
**Model = Object trong code Python tương ứng với một bảng trong database.**

# 📘 **MEETMATE DATABASE SCHEMA DOCUMENTATION**

**Version:** 1.0  
**Engine:** PostgreSQL + pgvector  
**Purpose:** AI-powered Meeting Intelligence System (Pre → In → Post meeting)

* * *

# 1. Overview

MeetMate là hệ thống hỗ trợ họp thông minh gồm ba pha:

1. **Pre-meeting**  
    Sinh agenda, tài liệu, gợi ý người cần mời, thu câu hỏi.
    
2. **In-meeting**  
    ASR transcript, live recap, action/decision/risk mining, Ask-AI, tool-calling.
    
3. **Post-meeting**  
    Sinh minutes, đồng bộ tasks, reminders, highlights, Q&A từ transcript.
    

Database schema được thiết kế để:

* **Audit-friendly**: truy vết mọi quyết định trong transcript
    
* **AI-native**: hỗ trợ embedding, vector search, chunking
    
* **Scalable**: module hóa theo từng pha meeting
    
* **Multi-organization**: hỗ trợ nhiều công ty, phòng ban
    

* * *

# 2. Core Entities

(Organization, User, Project, Department)

MeetMate là multi-tenant nên tất cả user, project, department nằm dưới một organization.

## **2.1 Organization**

Quản lý tenant:

| Column | Description |
| --- | --- |
| id | UUID |
| name | Tên tổ chức |
| created_at | Thời điểm tạo |

## **2.2 Project**

Gắn meeting theo sản phẩm/dự án:

| Column | Description |
| --- | --- |
| organization_id | FK đến organization |
| name, code | Tên & mã dự án |

## **2.3 Department**

Phòng ban của user và meeting:

## **2.4 User Account**

User thuộc tổ chức, có thông tin cần thiết cho AI personalization.

**Vai trò:**

* user
    
* chair
    
* PMO
    
* admin
    

**Indexes:**

* `idx_user_email`
    
* `idx_user_org`
    

* * *

# 3. Meeting Core

## **3.1 Meeting**

Đối tượng trung tâm của hệ thống.

| Field | Description |
| --- | --- |
| external_event_id | ID Sync từ Outlook/Teams |
| organizer_id | Ai là host |
| phase | pre / in / post |
| meeting_type | risk / steering / sprint / daily |
| project_id, department_id | Liên kết tổ chức |

**Indexes giúp tối ưu truy vấn:**

* tổ chức, loại họp, thời gian
    

## **3.2 Meeting Participant**

Trạng thái RSVP + tracking join/leave.

| role | organizer / required / optional / attendee |

* * *

# 4. Pre-meeting Module

Mục tiêu: chuẩn bị agenda, đọc tài liệu, gom câu hỏi.

* * *

## **4.1 Agenda Proposed (AI-generated)**

Lưu agenda tự động.

| status | draft / approved / rejected |

AI dùng bảng này để:

* regenerate agenda
    
* track chỉnh sửa
    
* so sánh bản dự thảo với bản final
    

* * *

## **4.2 Preread Documents (RAG)**

Tài liệu AI gợi ý đọc trước.

| relevance_score | mức liên quan từ AI |  
| status | suggested / accepted / ignored |

* * *

## **4.3 Pre-meeting Questions**

Thu câu hỏi/risk từ người tham dự.

| type | question / risk / demo_request |

* * *

## **4.4 Meeting Suggestions**

AI gợi ý:

* người cần mời
    
* tài liệu cần đọc
    
* policy cần xem
    
* action cần chuẩn bị
    

* * *

# 5. In-meeting Module

Phần quan trọng nhất của MeetMate.

* * *

## 5.1 Transcript Chunk

Kết quả từ:

* ASR speech-to-text
    
* diarization: phân người nói
    
* segmentation: chia đoạn
    

**Lý do chia chunk:**

* Tăng tốc vector search
    
* Lưu metadata speaker
    
* Liên kết đến action/decision/risk
    
* Dễ crop video highlight
    

| Field | Description |
| --- | --- |
| chunk_index | thứ tự xuất hiện |
| start_time / end_time | giây từ đầu cuộc họp |
| speaker | speaker_0, speaker_1 |
| speaker_user_id | map user nếu nhận diện được |
| confidence | từ ASR |

**Indexes:**

* meeting_id
    
* meeting_id + start_time
    

* * *

## 5.2 Live Recap Snapshot

AI nén 1 nhóm chunk thành recap 1–3 phút.

* * *

## 5.3 Action / Decision / Risk Items

Ba bảng riêng để hỗ trợ:

* phân quyền
    
* sync task
    
* audit
    
* phân tích performance
    

**Mỗi item có nguồn gốc:**

* source_chunk_id
    
* source_text
    

→ Audit: "Action này được nói ở phút nào? Ai nói?"

**Action Fields:**

| Field | Description |
| --- | --- |
| owner_user_id | ai chịu trách nhiệm |
| deadline | hạn |
| status | proposed → confirmed → completed |

* * *

## 5.4 Ask-AI Query

Khi user hỏi trong họp:

* MeetMate RAG transcript + documents
    
* Lưu câu hỏi, câu trả lời, citation
    
* Lưu chunk context để giải thích vì sao AI trả lời vậy
    

* * *

## 5.5 Tool-calling Logs

Ghi lại các thao tác:

* Tạo task Planner
    
* Tạo lịch Outlook
    
* Mở tài liệu SharePoint
    

Giúp audit + debug workflow.

* * *

## 5.6 Agenda Progress Tracking

Phục vụ timeboxing và etiquette AI.

* * *

# 6. Post-meeting Module

* * *

## 6.1 Meeting Minutes

Minutes AI-generated.

| version | vì minutes có thể được chỉnh nhiều lần |  
| status | draft / reviewed / approved / distributed |

Hỗ trợ:

* exec summary
    
* HTML
    
* markdown
    

* * *

## 6.2 Minutes Distribution Log

Theo dõi việc gửi minutes qua:

* email
    
* Teams
    
* sharepoint
    

Hỗ trợ:

* trạng thái đọc
    
* failed retry
    

* * *

## 6.3 Task Sync Log

Đồng bộ action item sang:

* Jira
    
* MS Planner
    
* Asana
    

* * *

## 6.4 Deadline Reminder Log

Gửi nhắc hạn:

* 3 ngày trước deadline
    
* 1 ngày trước
    
* overdue
    

* * *

## 6.5 Highlight Clip

Phục vụ video summary:

| Field | Description |
| --- | --- |
| start_time | vị trí video |
| reason | vì sao chọn đoạn này |
| transcript_text | để hiển thị caption |

* * *

## 6.6 Post-meeting Q&A

Cho phép user truy vấn transcript sau họp.

* * *

# 7. Vector Store (RAG)

* * *

## 7.1 Document

Nguồn tài liệu tổ chức:

* SharePoint
    
* LOffice
    
* Upload
    
* Wiki
    

## 7.2 Document Embedding

Mỗi chunk 100–500 từ:

* embedding vector (1536)
    
* metadata: {page, heading, section}
    

Dùng cho:

* semantic search
    
* Ask-AI
    
* pre-read suggestions
    

* * *

## 7.3 Transcript Embedding

Phục vụ:

* search nội dung cuộc họp cũ
    
* build highlights
    

* * *

# 8. Chat & Session Management

* * *

## 8.1 Chat Session

Một session chat liên quan đến:

* meeting
    
* user
    
* loại context: pre/in/post
    

## 8.2 Chat Message

Lưu hội thoại:

* role (user/assistant/system)
    
* citations
    
* tool-calls
    

* * *

# 9. Audit & History

* * *

## 9.1 Minutes Edit History

Theo dõi chỉnh sửa từng trường minutes.

* * *

## 9.2 Item Confirmation Log

Ghi lại:

* Ai confirm action
    
* Trạng thái trước–sau
    
* Comment
    

* * *

# 10. Vector Index (pgvector IVFFlat)

Hai index tối ưu cho:

* semantic document search
    
* semantic transcript search
    

```sql
CREATE INDEX idx_docemb_vector ON document_embedding 
USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

* * *

# 11. Relationship Diagram (mô tả tổng quát)

* **organization** → 1-n → user_account
    
* **organization** → 1-n → project, department
    
* **meeting** → n-n → user_account (meeting_participant)
    
* **meeting** → 1-n → transcript_chunk
    
* **meeting** → 1-n → action/decision/risk
    
* **meeting** → 1-n → ask_ai_query
    
* **meeting** → 1-n → meeting_minutes
    
* **meeting_minutes** → 1-n → distribution_log
    
* **document** → 1-n → document_embedding
    
* **transcript_chunk** → 1-n → transcript_embedding
    

* * *

# 12. Summary

Database này được thiết kế để hỗ trợ 4 mục tiêu chính:

### **1. Meeting Intelligence**

* Action mining
    
* Decision tracking
    
* Timeboxing
    
* Transcript summarization
    

### **2. AI-powered RAG**

* Semantic document search
    
* Transcript Q&A
    
* Policy lookup
    

### **3. Audit & Compliance**

* Traceability hành động, quyết định
    
* Lịch sử chỉnh sửa minutes
    
* Logging tool-calling
    

### **4. Scalability**

* Multi-organization
    
* Vector store tách rời
    
* Module hóa theo pha meeting
    
