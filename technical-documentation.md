# Sibyl AI LINE Bot 技術實作文檔

**專案名稱**：五湖園生命智慧園區 AI LINEBot-Sibyl
**版本**：3.4  
**技術架構**：GPT-4o + Google Cloud Platform  
**文檔日期**：2025-07-18  
**開發狀態**：核心功能完成，雙模式系統穩定運行，條件性欄位實作完成，防濫用機制已上線

---

## 目錄

1. [系統架構設計](#系統架構設計)
2. [技術棧詳解](#技術棧詳解)
3. [核心模組詳解](#核心模組詳解)
4. [資料庫設計](#資料庫設計)
5. [API 設計與整合](#api-設計與整合)
6. [部署與 DevOps](#部署與-devops)
7. [開發環境設定](#開發環境設定)
8. [程式碼規範](#程式碼規範)
9. [運維監控](#運維監控)
10. [故障排除](#故障排除)
11. [安全性設計](#安全性設計)
12. [效能優化](#效能優化)

---

## 系統架構設計

### 整體架構圖

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   LINE Platform │◄──►│   Cloud Run      │◄──►│  External APIs  │
│                 │    │  (Sibyl AI Bot)  │    │                 │
│   - Webhook     │    │  - FastAPI       │    │  - OpenAI GPT-4o│
│   - Messaging   │    │  - Python 3.11   │    │  - Embeddings   │
│   - Users       │    │  - Docker        │    │                 │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                               │                          
                               ▼                          
                    ┌──────────────────┐         
                    │  Google Cloud    │         
                    │   Services       │         
                    │ - Firestore      │         
                    │ - Sheets API     │         
                    │ - Secret Manager │         
                    │ - Cloud Build    │         
                    │ - Artifact Reg   │         
                    └──────────────────┘         
```

### 資料流向詳解

#### 1. 訊息接收流程
```
LINE User → LINE Platform → Cloud Run Webhook → message_processor.py
```

#### 2. 模式判斷流程
```
User Message → user_mode_manager.py → Check Mode (Traditional/AI)
```

#### 3. 傳統模式流程
```
Traditional Mode → auto_reply_handler.py → Keyword Match → Reply
```

#### 4. AI 模式流程
```
AI Mode → intent_classifier.py → OpenAI GPT-4o → Intent Result
```

#### 5. 狀態管理流程
```
Intent Result → conversation_state.py → Firestore → State Update
```

#### 6. 知識檢索流程
```
Query → retriever.py → Vector Search + Text Search → FAQ Match
```

#### 7. 回應生成流程
```
Context + Query → prompt_builder.py → GPT-4o → Response Generation
```

#### 8. 記憶儲存流程
```
Conversation → memory_manager.py → Firestore → Long-term Storage
```

#### 9. 日誌記錄流程
```
All Interactions → sheets_repo.py → Google Sheets → Analytics
```

### 架構設計原則

#### 微服務架構
- **單一職責**：每個模組負責特定功能
- **鬆散耦合**：模組間透過明確介面通信
- **高內聚**：相關功能聚集在同一模組

#### 雲端原生設計
- **容器化部署**：Docker + Cloud Run
- **自動擴展**：基於流量自動調整資源
- **狀態分離**：應用邏輯與資料分離

#### 錯誤處理策略
- **優雅降級**：AI 失敗時回退到 FAQ
- **人工接管**：複雜問題自動轉接
- **即時恢復**：系統錯誤時自動重試

---

## 技術棧詳解

### 後端框架

#### FastAPI 選擇理由
```python
FastAPI 優勢:
- 高效能: 基於 Starlette + Pydantic
- 自動文檔: OpenAPI/Swagger 自動生成
- 類型安全: 完整的 Type Hints 支援
- 異步支援: Native async/await
- 快速開發: 直觀的 API 設計
```

#### 核心依賴
```python
# requirements.txt 關鍵套件
fastapi==0.111.0          # Web 框架
uvicorn[standard]==0.30.0  # ASGI 伺服器
line-bot-sdk>=3.0.0        # LINE Bot SDK
openai==1.25.0             # OpenAI API
gspread==6.0.2             # Google Sheets
google-auth                # Google 認證
numpy==1.24.3              # 向量運算
```

### AI 與機器學習

#### OpenAI GPT-4o 配置
```python
# app/config.py
MODEL_NAME = "gpt-4o"
MODEL_TEMPERATURE = 0.85
MODEL_EMBED = "text-embedding-3-large"

# 成本設定 (USD per 1K tokens)
COST_INPUT = 0.0010
COST_OUTPUT = 0.0030

# 客戶端初始化
openai_client = openai.OpenAI(api_key=OPENAI_API_KEY)
```

#### Embedding 向量處理
```python
# 向量維度與分割
EMBEDDING_DIMENSIONS = 3072  # text-embedding-3-large
EMBEDDING_SPLIT_SIZE = 1536  # 分割為兩部分儲存

# 餘弦相似度計算
def cosine_similarity(vec1, vec2):
    dot_product = np.dot(vec1, vec2)
    norm1 = np.linalg.norm(vec1)
    norm2 = np.linalg.norm(vec2)
    return dot_product / (norm1 * norm2)
```

### 資料儲存架構

#### Firestore (NoSQL) 設計
```javascript
// 集合結構
Collections:
- conversation_states     // 對話狀態管理
- recent_conversations   // 30天短期記憶
- long_term_memory      // 永久摘要記憶

// 文檔 TTL 設定
TTL_CONVERSATION_STATES = 7 days
TTL_RECENT_CONVERSATIONS = 30 days
TTL_LONG_TERM_MEMORY = permanent
```

#### Google Sheets 資料管理
```yaml
Worksheets:
- FAQ:              問答知識庫 + 向量儲存
- settings:         系統參數配置
- chat_log:         對話記錄分析
- manual_override:  人工接管管理
- holidays:         假期設定
- admins:           管理員清單
- 原始自動回覆:      傳統模式自動回覆設定
```

### 部署平台配置

#### Google Cloud Platform
```yaml
Project Configuration:
- Project ID: bright-gearbox-462803-f5
- Region: asia-east1 (台灣)
- Service Account: 487710707313-compute@developer.gserviceaccount.com

Cloud Run Settings:
- CPU: 2 vCPU
- Memory: 2Gi
- Min Instances: 1
- Max Instances: 10
- Concurrency: 50
- Request Timeout: 300s
```

#### 環境變數管理
```bash
# 必要環境變數
SHEET_ID=1_4R_ftouDHhUXygW3ab7YlEzLWbXytwudfOegUIhu5Y
MODEL_NAME=gpt-4o
MODEL_EMBED=text-embedding-3-large
MODEL_TEMPERATURE=0.85

# 檢索參數
CONTEXT_K=5
SIM_THRESHOLD=0.50
VECTOR_SIM_THRESHOLD=0.60
WEIGHT_PRIORITY=0.10

# 快取設定
FAQ_CACHE_TTL=3600
```

---

## 核心模組詳解

### 1. 訊息處理器 (message_processor.py)

#### 責任鏈設計模式
```python
class MessageProcessor:
    async def process_message(self, event: dict) -> None:
        """主要處理流程"""
        user_id = event["source"]["userId"]
        user_msg = event["message"]["text"]
        reply_token = event["replyToken"]
        
        # 1. 管理指令檢查
        if manual_override_manager.is_admin_command(user_msg):
            manual_override_manager.handle_admin_command(user_id, user_msg)
            return
        
        # 2. 人工接管狀態檢查
        if manual_override_manager.is_in_manual_mode(user_id):
            return
        
        # 3. 對話記憶儲存
        await memory_manager.save_message(user_id, Message(...))
        
        # 4. 主要業務邏輯處理
        await self._handle_user_message(user_id, user_msg, reply_token)
```

#### 雙模式處理邏輯
```python
async def _handle_user_message(self, user_id: str, user_msg: str, reply_token: str):
    """處理使用者訊息的主要邏輯"""
    # 檢查用戶模式
    user_mode = await self.user_mode_manager.get_user_mode(user_id)
    
    if user_mode == "traditional":
        # 傳統模式處理
        
        # 1. 優先檢查一律回應
        always_reply = self.auto_reply_handler.find_always_reply()
        if always_reply:
            self._send_auto_reply(always_reply, user_id, user_msg, reply_token, start_time)
            return
            
        # 2. 檢查是否要切換到 AI 模式
        if user_msg.strip() in ["0", "AI客服", "智慧客服", "ai客服", "AI", "ai"]:
            await self.user_mode_manager.set_user_mode(user_id, "ai")
            # 發送 AI 歡迎訊息
            return
            
        # 3. 嘗試自動回覆匹配
        auto_reply = self.auto_reply_handler.find_auto_reply(user_msg)
        if auto_reply:
            self._send_auto_reply(auto_reply, user_id, user_msg, reply_token, start_time)
            return
            
        # 4. 無匹配 - 顯示引導訊息
        guide_message = "抱歉，我不太理解您的意思。\\n\\n您可以：\\n1️⃣ 輸入數字 1-9 選擇服務\\n0️⃣ 輸入 0 使用 AI 智慧客服"
        reply_to_line(reply_token, TextMessage(text=guide_message))
        
    else:  # AI 模式
        # AI 模式現在是永久的，不需要更新活動時間
        
        # 檢查是否要返回傳統模式
        if user_msg.strip() in ["返回", "返回選單", "選單", "回選單", "主選單"]:
            await self.user_mode_manager.set_user_mode(user_id, "traditional")
            # 顯示傳統選單提示
            return
            
        # 繼續原有的 AI 處理邏輯...
```

#### 處理流程詳解
```python
async def _handle_user_message(self, user_id, user_msg, reply_token):
    """主要業務邏輯"""
    # 取得對話記憶
    memory_context = await memory_manager.format_context_for_prompt(user_id)
    
    # 意圖識別
    intent_result = await intent_classifier.classify(user_msg, [])
    
    # 檢查進行中的對話狀態
    active_state = await state_manager.get_active_state(user_id)
    
    if active_state:
        # 繼續多輪對話
        await self._handle_conversation_flow(user_id, user_msg, reply_token, active_state)
    else:
        # 根據意圖路由
        await self._route_by_intent(intent_result, user_id, user_msg, reply_token, memory_context)
```

### 2. 自動回覆處理器 (auto_reply_handler.py)

#### 功能概述
處理傳統模式下的精確關鍵字匹配，支援一律回應和可點擊圖片訊息。

#### 主要功能
```python
class AutoReplyHandler:
    def find_auto_reply(self, message: str) -> Optional[dict]:
        """精確匹配用戶訊息"""
        auto_replies = load_auto_replies()
        clean_message = message.strip()
        
        for reply in auto_replies:
            keywords = reply.get('keywords', [])
            for keyword in keywords:
                if clean_message == keyword.strip():
                    return reply
        return None
    
    def find_always_reply(self) -> Optional[dict]:
        """尋找一律回應設定"""
        auto_replies = load_auto_replies()
        
        for reply in auto_replies:
            if reply.get('is_always_reply', False) and reply.get('title', '').strip() == '自動回應':
                return reply
        return None
    
    def create_clickable_image_template(self, reply_data: dict) -> Optional['TemplateMessage']:
        """建立可點擊的圖片訊息（ButtonsTemplate）"""
        image_url = reply_data.get('template_image_url', '').strip()
        action_url = reply_data.get('template_action_url', '').strip()
        
        if not image_url or not action_url:
            return None
            
        uri_action = create_uri_action(action_label[:20], action_url)
        template = create_buttons_template(
            title=reply_data.get('title', '')[:40],
            text=reply_data.get('template_text', '點擊圖片查看詳情')[:60],
            image_url=image_url,
            actions=[uri_action],
            alt_text=reply_data.get('title', '請查看圖片')
        )
        return template
```

### 3. 用戶模式管理器 (user_mode_manager.py)

#### 功能概述
管理用戶在傳統模式和 AI 模式之間的切換，支援永久 AI 模式。

#### 主要功能
```python
class UserModeManager:
    async def get_user_mode(self, user_id: str) -> str:
        """取得用戶模式，預設為 'traditional'"""
        # 不再檢查超時，AI 模式永久保持
        doc_ref = self.db.collection(self.collection_name).document(user_id)
        doc = doc_ref.get()
        
        if doc.exists:
            data = doc.to_dict()
            return data.get('mode', 'traditional')
        return 'traditional'
    
    async def set_user_mode(self, user_id: str, mode: str):
        """設定用戶模式"""
        doc_ref = self.db.collection(self.collection_name).document(user_id)
        doc_ref.set({
            'mode': mode,
            'last_activity': datetime.utcnow(),
            'updated_at': datetime.utcnow()
        }, merge=True)
```

### 4. 意圖分類器 (intent_classifier.py)

#### 意圖類型定義
```python
class IntentType(Enum):
    PRICE_INQUIRY = "price_inquiry"           # 價格詢問
    EVENT_REGISTRATION = "event_registration" # 法會報名
    ACTIVATION_SERVICE = "activation_service" # 塔位/牌位啟用
    RITUAL_SERVICE = "ritual_service"         # 佛事登記
    VISIT_BOOKING = "visit_booking"           # 參觀預約
    GENERAL_INQUIRY = "general_inquiry"       # 一般諮詢

@dataclass
class IntentResult:
    primary_intent: IntentType
    confidence: float
    extracted_data: Dict[str, any] = None
    sub_type: str = None
    need_human: bool = False
    reason: str = ""
    emotion_tags: List[str] = None  # 新增：情緒標籤
```

#### GPT-4o 驅動的分類
```python
class LLMIntentClassifier:
    def __init__(self):
        self.model = MODEL_NAME
        self.system_prompt = self._build_system_prompt()
    
    async def classify(self, message: str, context: List[Dict] = None) -> IntentResult:
        response = openai_client.chat.completions.create(
            model=self.model,
            messages=[
                {"role": "system", "content": self.system_prompt},
                {"role": "user", "content": self._build_user_prompt(message, context)}
            ],
            temperature=0.3,
            response_format={"type": "json_object"}
        )
        
        result = json.loads(response.choices[0].message.content)
        return IntentResult(
            primary_intent=IntentType(result["intent"]),
            confidence=result["confidence"],
            extracted_data=result.get("extracted_data", {}),
            sub_type=result.get("sub_type"),
            need_human=result.get("need_human", False),
            reason=result.get("reason", "")
        )
```

### 3. 對話狀態管理 (conversation_state.py)

#### 狀態資料結構
```python
@dataclass
class ConversationState:
    user_id: str
    current_intent: str
    current_step: int
    collected_data: Dict[str, any]
    field_sequence: List[str]
    started_at: datetime
    updated_at: datetime
    expires_at: datetime
    # 新增：待確認意圖相關欄位
    pending_intent: Optional[str] = None
    pending_data: Optional[Dict[str, any]] = None
    
    def to_dict(self) -> dict:
        return {
            "user_id": self.user_id,
            "current_intent": self.current_intent,
            "current_step": self.current_step,
            "collected_data": self.collected_data,
            "field_sequence": self.field_sequence,
            "started_at": self.started_at,
            "updated_at": self.updated_at,
            "expires_at": self.expires_at,
            "pending_intent": self.pending_intent,
            "pending_data": self.pending_data
        }
```

#### 條件性欄位流程定義
```python
class ConversationFlows:
    # 法會報名流程（包含條件性欄位）
    EVENT_REGISTRATION = {
        "fields": ["陽世報名人", "往生菩薩", "報名項目", "電話", "地址"],
        "prompts": {
            "陽世報名人": "請問陽世報名人的姓名是？（請以一人做代表）",
            "往生菩薩": "請問要為哪位往生菩薩報名呢？",
            "報名項目": "請問要報名哪個法會項目？",
            "電話": "請提供您的聯絡電話：",
            "地址": "請提供您的地址：",
            "拜桌處理方式": "請問拜桌要如何處理？\n1️⃣ 代捐出公益單位\n2️⃣ 自行取回\n請回覆 1 或 2",
            "收件人": "請提供收件人姓名（可回覆「同上」使用報名人資料）：",
            "收件人電話": "請提供收件人電話（可回覆「同上」）：",
            "收件地址": "請提供收件地址（可回覆「同上」）："
        },
        "conditional_fields": {
            "報名項目": {
                "拜桌": ["拜桌處理方式"]
            },
            "拜桌處理方式": {
                "託運寄送": ["收件人", "收件人電話", "收件地址"]
            }
        }
    }
    
    # 佛事登記流程（包含條件性欄位）
    RITUAL_SERVICE = {
        "fields": ["申請人", "先靈姓名", "位置編號", "佛事項目", "佛事日期", "佛事時間"],
        "prompts": {
            "申請人": "請問申請人的姓名是？",
            "先靈姓名": "請問先靈的姓名是？",
            "位置編號": "請提供塔位或牌位編號：",
            "佛事項目": "請問要辦理什麼佛事？",
            "佛事日期": "請問希望的佛事日期？（例如：12/25）",
            "佛事時間": "請問希望的時間？（上午/下午）",
            "對年處理方式": "請選擇對年處理方式：\n1️⃣ 單純誦經做對年\n2️⃣ 請香火回家\n3️⃣ 在塔上合爐\n請回覆 1、2 或 3"
        },
        "conditional_fields": {
            "佛事項目": {
                "對年": ["對年處理方式"]
            }
        }
    }
```

#### 動態欄位管理器
```python
class ConversationStateManager:
    def __init__(self):
        self.collection = db.collection('conversation_states')
        self.flows = ConversationFlows()
    
    def get_dynamic_fields(self, state: ConversationState) -> List[str]:
        """取得動態欄位列表（包含條件性欄位）"""
        flow = self.get_flow_for_intent(state.current_intent)
        base_fields = flow["fields"].copy()
        conditional_fields = flow.get("conditional_fields", {})
        
        # 遞迴處理條件性欄位
        fields_to_add = []
        
        def process_conditional_fields(field_name: str, field_value: str):
            if field_name in conditional_fields:
                conditions = conditional_fields[field_name]
                if field_value in conditions:
                    new_fields = conditions[field_value]
                    for new_field in new_fields:
                        if new_field not in fields_to_add:
                            fields_to_add.append(new_field)
                        # 遞迴處理多層條件
                        if new_field in state.collected_data:
                            process_conditional_fields(new_field, state.collected_data[new_field])
        
        # 檢查所有已收集的資料
        for field, value in state.collected_data.items():
            process_conditional_fields(field, value)
        
        # 將條件性欄位插入到適當位置
        if fields_to_add:
            # 找到觸發條件欄位的位置
            for field in fields_to_add:
                if field not in base_fields:
                    base_fields.append(field)
        
        return base_fields
    
    def get_next_field(self, state: ConversationState) -> Optional[str]:
        """取得下一個要收集的欄位"""
        dynamic_fields = self.get_dynamic_fields(state)
        
        for field in dynamic_fields:
            if field not in state.collected_data:
                return field
        
        return None
    
    async def create_new_state(self, user_id: str, intent: str, initial_data: Dict = None):
        flow = self.get_flow_for_intent(intent)
        now = datetime.now(timezone.utc)
        
        state = ConversationState(
            user_id=user_id,
            current_intent=intent,
            current_step=0,
            collected_data=initial_data or {},
            field_sequence=flow["fields"],  # 基礎欄位
            started_at=now,
            updated_at=now,
            expires_at=now + timedelta(days=7)
        )
        
        await self.save_state(state)
        return state
```

### 4. 記憶管理器 (memory_manager.py)

#### 雙層記憶架構
```python
class MemoryManager:
    async def save_message(self, user_id: str, message: Message, context: ConversationContext = None):
        """儲存訊息到短期記憶"""
        doc_ref = self.db.collection(RECENT_CONVERSATIONS).document(user_id)
        
        current_time = datetime.now(timezone.utc)
        expire_time = current_time + timedelta(days=SHORT_TERM_DAYS)
        
        # 更新或建立記憶文件
        if doc.exists:
            data = doc.to_dict()
            messages = data.get("messages", [])
            messages.append(message.to_dict())
            doc_ref.update({
                "messages": messages,
                "last_activity": current_time,
                "expires_at": expire_time
            })
        else:
            data = {
                "user_id": user_id,
                "messages": [message.to_dict()],
                "context": context.to_dict() if context else {},
                "created_at": current_time,
                "last_activity": current_time,
                "expires_at": expire_time
            }
            doc_ref.set(data)
```

#### 記憶格式化
```python
async def format_context_for_prompt(self, user_id: str) -> str:
    """格式化記憶供 GPT 使用"""
    recent_messages = await self.get_recent_messages(user_id)
    context = await self.get_conversation_context(user_id)
    long_term = await self.get_long_term_memory(user_id)
    
    prompt_parts = []
    
    # 長期記憶摘要
    if long_term and "important_people" in long_term:
        prompt_parts.append("【記得的重要人物】")
        for name, info in long_term["important_people"].items():
            relationship = info.get("relationship", "")
            memories = ", ".join(info.get("key_memories", []))
            prompt_parts.append(f"- {name}（{relationship}）：{memories}")
    
    # 最近對話
    if recent_messages:
        prompt_parts.append("\n【最近的對話】")
        for msg in recent_messages[-10:]:
            role = "客戶" if msg.role == "user" else "你"
            content = msg.content[:100] + "..." if len(msg.content) > 100 else msg.content
            prompt_parts.append(f"{role}：{content}")
    
    return "\n".join(prompt_parts) if prompt_parts else ""
```

### 5. 欄位驗證器 (field_validator.py)

#### 功能概述
提供即時的欄位格式驗證和錯誤提示，支援條件性欄位和隱藏選項處理。

#### 主要功能
```python
class FieldValidator:
    """欄位驗證器"""
    
    def validate_field(self, field_name: str, value: str, context: dict = None) -> Tuple[bool, str, Any]:
        """
        驗證欄位值
        
        Returns:
            (是否通過, 錯誤訊息, 格式化後的值)
        """
        # 檢查是否使用「同上」
        if value.strip().lower() in ["同上", "一樣", "相同", "同", "same"]:
            return self._handle_same_as_above(field_name, context)
        
        # 根據欄位名稱調用對應的驗證方法
        validators = {
            "電話": self.validate_phone,
            "收件人電話": self.validate_phone,
            "拜桌處理方式": self.validate_table_option,
            "對年處理方式": self.validate_year_option,
            # ... 更多驗證器
        }
        
        validator = validators.get(field_name, self.validate_generic)
        return validator(value, context)
```

#### 隱藏選項處理
```python
def validate_table_option(self, value: str, context: dict = None) -> Tuple[bool, str, str]:
    """驗證拜桌處理方式"""
    # 檢查是否主動要求託運（隱藏選項）
    shipping_keywords = ["託運", "寄送", "寄回", "宅配", "貨運", "運送", "郵寄"]
    if any(keyword in value for keyword in shipping_keywords):
        return True, "", "託運寄送"
    
    # 標準選項處理
    standard_mappings = {
        "1": "代捐出公益單位",
        "2": "自行取回",
        "代捐": "代捐出公益單位",
        "自取": "自行取回"
    }
    
    if value in standard_mappings:
        return True, "", standard_mappings[value]
    
    # 驗證失敗時，只顯示兩個選項（不顯示託運）
    error_msg = "請選擇拜桌處理方式：\n"
    error_msg += "1️⃣ 代捐出公益單位\n"
    error_msg += "2️⃣ 自行取回\n"
    error_msg += "請回覆 1 或 2"
    
    return False, error_msg, value
```

#### 同上功能實現
```python
def _handle_same_as_above(self, field_name: str, context: dict) -> Tuple[bool, str, Any]:
    """處理「同上」的情況"""
    # 對應關係
    mapping = {
        "收件人": "陽世報名人",
        "收件人電話": "電話",
        "收件地址": "地址"
    }
    
    source_field = mapping.get(field_name)
    if source_field and context and source_field in context:
        source_value = context[source_field]
        return True, "", source_value
    
    # 找不到對應資料
    return False, f"找不到可參照的{field_name}資料，請直接輸入", ""
```

### 6. 進度格式化器 (progress_formatter.py)

#### 功能概述
提供清晰的進度顯示和引導，支援動態欄位計算和條件性欄位顯示。

#### 動態進度計算
```python
class ProgressFormatter:
    """進度格式化器"""
    
    def format_progress(self, state: ConversationState, error_field: str = None) -> str:
        """格式化進度顯示"""
        # 取得總欄位數（包含條件性欄位）
        total_fields = self._get_total_fields(state)
        completed = len(state.collected_data)
        remaining = total_fields - completed
        
        # 建立進度條
        progress_bar = self._create_progress_bar(completed, total_fields)
        
        # 顯示已收集和待填寫的資料
        message = f"{self.EMOJI['progress']} 【{flow_name}進度】\n"
        message += f"{progress_bar}\n"
        message += f"已完成 {completed}/{total_fields} 項\n\n"
        
        return message
    
    def _get_total_fields(self, state: ConversationState) -> int:
        """計算總欄位數（包含多層條件性欄位）"""
        from app.conversation_state import get_state_manager
        state_manager = get_state_manager()
        
        # 使用動態欄位列表
        dynamic_fields = state_manager.get_dynamic_fields(state)
        return len(dynamic_fields)
```

#### 完成摘要格式化
```python
def _format_event_registration_summary(self, data: Dict) -> str:
    """格式化法會報名摘要"""
    # 基本報名資訊
    summary = f"📿 報名項目：{data.get('報名項目', '')}\n"
    summary += f"👤 陽世報名人：{data.get('陽世報名人', '')}\n"
    summary += f"🙏 往生菩薩：{data.get('往生菩薩', '')}\n"
    summary += f"📞 聯絡電話：{data.get('電話', '')}\n"
    summary += f"🏠 聯絡地址：{data.get('地址', '')}\n"
    
    # 拜桌處理方式
    if data.get('報名項目') == '拜桌' and '拜桌處理方式' in data:
        summary += f"📦 拜桌處理：{data['拜桌處理方式']}"
        
        # 託運資訊放在最後
        if data['拜桌處理方式'] == '託運寄送':
            summary += f"\n\n【託運資訊】\n"
            summary += f"👤 收件人：{data.get('收件人', '')}\n"
            summary += f"📞 收件電話：{data.get('收件人電話', '')}\n"
            summary += f"📍 收件地址：{data.get('收件地址', '')}"
    
    return summary
```

### 7. 檢索系統 (retriever.py)

#### 混合檢索架構
```python
def retrieve(user_text: str) -> Tuple[Optional[dict], float]:
    """混合檢索主函數"""
    faq_list = load_faq()
    result = None
    
    # 優先使用向量檢索
    if USE_VECTOR_SEARCH:
        result = _vector_search(user_text, faq_list)
        
        # 向量檢索失敗則退回文字檢索
        if not result:
            result = _text_search(user_text, faq_list)
    else:
        result = _text_search(user_text, faq_list)
    
    if result:
        return result.item, round(result.score, 2)
    
    return None, 0.0
```

#### 向量檢索實現
```python
def _vector_search(query: str, faq_list: List[dict]) -> Optional[SearchResult]:
    """向量語義檢索"""
    query_vec = _get_embedding(query)
    if query_vec is None:
        return None
    
    best_result = None
    best_score = 0.0
    
    for row in faq_list:
        # 從分割向量重建完整向量
        faq_vec = _parse_split_embedding(row)
        if faq_vec is None:
            continue
        
        # 計算相似度
        similarity = _cosine_similarity(query_vec, faq_vec)
        
        # 加入優先權重
        weighted_score = similarity * (1 + (row["priority"] - 1) * WEIGHT_PRIORITY)
        
        if weighted_score > best_score:
            best_score = weighted_score
            best_result = SearchResult(
                item=row,
                score=weighted_score,
                method="vector"
            )
    
    return best_result if best_result and best_result.score >= VECTOR_SIM_THRESHOLD else None
```

#### 向量分割處理
```python
def _parse_split_embedding(row: dict) -> Optional[np.ndarray]:
    """重建分割儲存的向量"""
    try:
        part1_str = row.get("embedding_part1", "").strip()
        part2_str = row.get("embedding_part2", "").strip()
        
        if not part1_str or not part2_str:
            return None
        
        part1 = json.loads(part1_str)
        part2 = json.loads(part2_str)
        full_vector = part1 + part2
        
        return np.array(full_vector)
    except (json.JSONDecodeError, ValueError, TypeError) as e:
        logging.warning(f"Error parsing split embedding: {e}")
        return None
```

---

## 資料庫設計

### Firestore 集合設計

#### conversation_states 集合
```javascript
// 文檔結構
{
  "user_id": "LINE_USER_ID",
  "current_intent": "event_registration",
  "current_step": 2,
  "collected_data": {
    "陽世報名人": "王小明",
    "往生菩薩": "王老太太"
  },
  "field_sequence": ["陽世報名人", "往生菩薩", "報名項目", "電話", "地址"],
  "started_at": "2025-07-10T10:00:00Z",
  "updated_at": "2025-07-10T10:05:00Z",
  "expires_at": "2025-07-17T10:00:00Z"
}

// 索引設計
- user_id (單一欄位索引)
- expires_at (TTL 索引)
```

#### recent_conversations 集合
```javascript
// 文檔結構
{
  "user_id": "LINE_USER_ID",
  "messages": [
    {
      "role": "user",
      "content": "我想了解塔位價格",
      "timestamp": "2025-07-10T10:00:00Z",
      "emotion_tags": ["inquiry"]
    },
    {
      "role": "assistant", 
      "content": "塔位價格8萬元起...",
      "timestamp": "2025-07-10T10:00:03Z"
    }
  ],
  "context": {
    "mentioned_people": {
      "媽媽": "母親"
    },
    "key_events": ["價格詢問"],
    "emotional_state": "平靜詢問"
  },
  "created_at": "2025-07-10T10:00:00Z",
  "last_activity": "2025-07-10T10:00:03Z",
  "expires_at": "2025-08-09T10:00:00Z"  // 30天後過期
}

// 索引設計
- user_id (單一欄位索引)
- expires_at (TTL 索引)
- last_activity (排序索引)
```

#### long_term_memory 集合
```javascript
// 文檔結構
{
  "user_id": "LINE_USER_ID",
  "important_people": {
    "王老太太": {
      "relationship": "母親",
      "key_memories": ["詢問塔位價格", "2024年往生"],
      "last_mentioned": "2025-07-10T10:00:00Z"
    }
  },
  "emotional_journey": [
    {
      "date": "2025-07-10",
      "summary": "初次詢問塔位，情緒平穩",
      "dominant_emotion": "平靜"
    }
  ],
  "service_history": [
    {
      "date": "2025-07-10",
      "service_type": "price_inquiry",
      "completed": true
    }
  ],
  "created_at": "2025-07-10T10:00:00Z",
  "updated_at": "2025-07-10T10:00:03Z"
}

// 索引設計
- user_id (單一欄位索引)
- updated_at (排序索引)
```

### Google Sheets 資料結構

#### FAQ 工作表
```
欄位設計:
A: id                   // 唯一識別碼
B: category            // 分類
C: question            // 主要問題
D: variants            // 變體問法 (換行分隔)
E: answer              // 回答內容
F: media_urls          // 媒體連結 (換行分隔)
G: priority            // 優先級 (1-5)
H: is_active           // 是否啟用 (TRUE/FALSE)
I: valid_from          // 生效日期
J: valid_to            // 失效日期
K: embedding_part1     // 向量前半部 (JSON)
L: embedding_part2     // 向量後半部 (JSON)
M: embedding_ts        // 向量生成時間
N: updated_at          // 最後更新時間
```

#### settings 工作表
```
欄位設計:
A: key                 // 設定鍵
B: value               // 設定值

範例內容:
prompt_template        // 提示詞模板
sim_threshold         // 相似度門檻
vector_sim_threshold  // 向量相似度門檻
use_vector_search     // 是否使用向量檢索
manual_override_ttl_hours // 人工接管TTL小時數
```

#### chat_log 工作表
```
欄位設計:
A: timestamp          // 時間戳記
B: user_id           // 使用者ID
C: user_message      // 使用者訊息
D: matched_id        // 匹配的FAQ ID
E: score             // 相似度分數
F: bot_response      // 機器人回應
G: media             // 媒體連結
H: model             // 使用的AI模型
I: prompt_tokens     // 輸入token數
J: completion_tokens // 輸出token數
K: cost_usd          // 成本(美元)
L: response_time_ms  // 回應時間(毫秒)
```

#### 原始自動回覆工作表
```
欄位設計:
A: 標題               // 顯示名稱
B: 內容               // 回覆文字內容（支援 [split] 分隔）
C: 回應類型           // "關鍵字回應:" 或 "一律回應"
D: 回應圖片1-4        // 圖片 URL（最多 4 張）
H: 模板圖片           // 可點擊圖片 URL
I: 點擊連結           // 點擊後導向的 HTTPS 網址
J: 按鈕文字           // 按鈕顯示文字（最多 20 字）
K: 模板說明           // 圖片下方說明（最多 60 字）

特殊設定:
- 一律回應：標題為「自動回應」且回應類型包含「一律回應」
- 關鍵字格式：在「關鍵字回應:」後換行分隔多個關鍵字
- 圖片直鏈：回應圖片1-4 支援 HTTPS 圖片直鏈（imgur、圖床、CDN 等）

圖片要求:
- 必須使用 HTTPS 協定
- 必須為公開可存取的直接連結
- 支援格式：JPEG、PNG
- 檔案大小：最大 10MB
- 圖片尺寸：最大 1024x1024 像素
```

---

## API 設計與整合

### FastAPI 路由設計

#### 主要端點
```python
# app/main.py
@app.get("/")
async def root():
    """健康檢查端點"""
    return {
        "service": "Sibyl AI LINE Bot",
        "status": "active",
        "version": "3.0",
        "model": MODEL_NAME
    }

@app.get("/health")
async def health_check():
    """詳細健康檢查"""
    health_status = {
        "status": "healthy",
        "checks": {
            "openai": False,
            "sheets": False,
            "firestore": False
        }
    }
    # 檢查各服務狀態...
    return health_status

@app.post("/webhook")
async def webhook(req: Request):
    """LINE Webhook 端點"""
    return await handle_line_webhook(req)
```

#### Webhook 處理流程
```python
# app/line_handler.py
async def handle_line_webhook(request: Request):
    """處理 LINE webhook 請求"""
    # 1. 驗證簽名
    signature = request.headers.get('X-Line-Signature')
    body_bytes = await request.body()
    
    if not _verify_signature(body_bytes, signature):
        raise HTTPException(status_code=403, detail="Invalid signature")
    
    # 2. 解析請求
    body_json = json.loads(body_bytes.decode('utf-8'))
    
    # 3. 檢查人工接管TTL
    manual_override_manager.check_ttl()
    
    # 4. 處理事件
    events = body_json.get("events", [])
    tasks = []
    
    for event in events:
        if event.get("type") == "message" and event["message"].get("type") == "text":
            task = asyncio.create_task(_process_event(event))
            tasks.append(task)
    
    if tasks:
        await asyncio.gather(*tasks, return_exceptions=True)
    
    return {"status": "ok"}
```

### LINE Bot SDK 整合

#### 訊息回覆封裝
```python
# app/line_utils.py
from linebot.v3.messaging import (
    ApiClient, MessagingApi, Configuration,
    ReplyMessageRequest, TextMessage, ImageMessage
)

# 初始化 LINE Bot API
configuration = Configuration(access_token=LINE_CHANNEL_ACCESS_TOKEN)
api_client = ApiClient(configuration)
line_bot_api = MessagingApi(api_client)

def reply_to_line(reply_token: str, messages: Union[Message, List[Message]]) -> bool:
    """回覆訊息到 LINE"""
    if not isinstance(messages, list):
        messages = [messages]
    
    # LINE 限制最多 5 則訊息
    if len(messages) > 5:
        messages = messages[:5]
    
    try:
        reply_request = ReplyMessageRequest(
            reply_token=reply_token,
            messages=messages
        )
        line_bot_api.reply_message(reply_request)
        return True
    except Exception as e:
        logger.error(f"Failed to reply to LINE: {e}")
        return False
```

#### 多媒體訊息處理
```python
def create_text_messages(text: str, split_token: str = "[split]") -> List[TextMessage]:
    """建立文字訊息，支援分割"""
    if not text:
        return []
    
    text_parts = text.split(split_token)
    messages = []
    
    for part in text_parts:
        part = part.strip()
        if part:
            messages.append(TextMessage(text=part))
    
    return messages

def combine_messages(text: str = None, image_urls: List[str] = None) -> List[Message]:
    """組合文字和圖片訊息"""
    messages = []
    
    if text:
        messages.extend(create_text_messages(text))
    
    if image_urls:
        for url in image_urls:
            if url and url.strip():
                messages.append(ImageMessage(
                    original_content_url=url.strip(),
                    preview_image_url=url.strip()
                ))
    
    return messages[:5]  # LINE 限制
```

### OpenAI API 整合

#### 客戶端配置
```python
# app/config.py
import openai

# 初始化客戶端
openai_client = openai.OpenAI(api_key=OPENAI_API_KEY)

# GPT-4o 調用封裝
async def call_gpt4o(messages: List[dict], temperature: float = MODEL_TEMPERATURE) -> dict:
    """GPT-4o API 調用封裝"""
    try:
        response = openai_client.chat.completions.create(
            model=MODEL_NAME,
            messages=messages,
            temperature=temperature,
            response_format={"type": "json_object"} if "json" in messages[0]["content"].lower() else None
        )
        return response
    except Exception as e:
        logger.error(f"OpenAI API error: {e}")
        raise
```

#### Embedding API 使用
```python
def get_embedding(text: str) -> Optional[np.ndarray]:
    """取得文字向量"""
    try:
        response = openai_client.embeddings.create(
            input=text,
            model=MODEL_EMBED
        )
        return np.array(response.data[0].embedding)
    except Exception as e:
        logger.error(f"Embedding API error: {e}")
        return None
```

### Google Cloud 服務整合

#### Secret Manager 整合
```python
# 金鑰管理
from google.cloud import secretmanager

def get_secret(secret_id: str, project_id: str) -> str:
    """從 Secret Manager 取得金鑰"""
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{project_id}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")
```

#### Firestore 連接
```python
# Firestore 客戶端
from google.cloud import firestore

# 使用 ADC (Application Default Credentials)
db = firestore.Client()

# 異步操作封裝
async def firestore_get(collection: str, document: str) -> dict:
    """異步取得文檔"""
    return await asyncio.to_thread(
        lambda: db.collection(collection).document(document).get().to_dict()
    )

async def firestore_set(collection: str, document: str, data: dict):
    """異步設定文檔"""
    await asyncio.to_thread(
        lambda: db.collection(collection).document(document).set(data)
    )
```

---

## 部署與 DevOps

### Docker 容器化

#### Dockerfile 設計
```dockerfile
FROM python:3.11-slim

# 設定工作目錄
WORKDIR /app

# 複製依賴文件
COPY requirements.txt .

# 安裝依賴套件
RUN pip install --no-cache-dir -r requirements.txt

# 複製應用程式程式碼
COPY ./app/ ./app

# 暴露端口
EXPOSE 8080

# 啟動指令
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

#### .dockerignore 設定
```
# .dockerignore
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.env
.git/
.gitignore
README.md
Dockerfile
.dockerignore
```

### Cloud Build CI/CD

#### cloudbuild.yaml 配置
```yaml
# cloudbuild.yaml
steps:
  # 取得 Git commit SHA
  - name: 'gcr.io/cloud-builders/git'
    id: 'Get commit SHA'
    entrypoint: 'sh'
    args:
    - '-c'
    - |
      SHORT_SHA=$(git rev-parse --short HEAD)
      echo $SHORT_SHA > /workspace/short_sha.txt

  # 建置 Docker 映像
  - name: 'gcr.io/cloud-builders/docker'
    id: 'Build Docker image'
    entrypoint: 'sh'
    args:
    - '-c'
    - |
      SHORT_SHA=$(cat /workspace/short_sha.txt)
      docker build \
        --no-cache \
        --tag asia-east1-docker.pkg.dev/bright-gearbox-462803-f5/linebot-repo/linebot:$SHORT_SHA \
        --tag asia-east1-docker.pkg.dev/bright-gearbox-462803-f5/linebot-repo/linebot:latest \
        .

  # 推送到 Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: 'Push Docker image'
    args:
    - 'push'
    - '--all-tags'
    - 'asia-east1-docker.pkg.dev/bright-gearbox-462803-f5/linebot-repo/linebot'

  # 部署到 Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'Deploy to Cloud Run'
    entrypoint: 'sh'
    args:
    - '-c'
    - |
      SHORT_SHA=$(cat /workspace/short_sha.txt)
      gcloud run deploy linebot \
        --image asia-east1-docker.pkg.dev/bright-gearbox-462803-f5/linebot-repo/linebot:$SHORT_SHA \
        --region asia-east1 \
        --platform managed \
        --allow-unauthenticated \
        --service-account 487710707313-compute@developer.gserviceaccount.com \
        --min-instances 1 \
        --max-instances 10 \
        --memory 2Gi \
        --cpu 2 \
        --timeout 300 \
        --concurrency 50 \
        --set-env-vars "SHEET_ID=1_4R_ftouDHhUXygW3ab7YlEzLWbXytwudfOegUIhu5Y,MODEL_NAME=gpt-4o,MODEL_EMBED=text-embedding-3-large,CONTEXT_K=5,FAQ_CACHE_TTL=3600,SIM_THRESHOLD=0.50,VECTOR_SIM_THRESHOLD=0.60,WEIGHT_PRIORITY=0.10,MODEL_TEMPERATURE=0.85" \
        --set-secrets "LINE_CHANNEL_ACCESS_TOKEN=line-channel-access-token:latest,LINE_CHANNEL_SECRET=line-channel-secret:latest,OPENAI_API_KEY=openai-api-key:latest"

options:
  logging: CLOUD_LOGGING_ONLY
```

### Cloud Run 服務配置

#### service.yaml 範例
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: linebot
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "10"
        autoscaling.knative.dev/minScale: "1"
    spec:
      serviceAccountName: 487710707313-compute@developer.gserviceaccount.com
      containers:
        - image: asia-east1-docker.pkg.dev/bright-gearbox-462803-f5/linebot-repo/linebot:latest
          ports:
            - containerPort: 8080
          env:
            # 從 Secret Manager 注入
            - name: LINE_CHANNEL_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: line-channel-access-token
                  key: "latest"
            - name: LINE_CHANNEL_SECRET
              valueFrom:
                secretKeyRef:
                  name: line-channel-secret
                  key: "latest"
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: openai-api-key
                  key: "latest"
            # 直接設定的環境變數
            - name: SHEET_ID
              value: "1_4R_ftouDHhUXygW3ab7YlEzLWbXytwudfOegUIhu5Y"
            - name: MODEL_NAME
              value: "gpt-4o"
            - name: MODEL_TEMPERATURE
              value: "0.85"
          resources:
            limits:
              memory: 2Gi
              cpu: "2"
      timeoutSeconds: 300
```

### 版本管理策略

#### Git 工作流程
```bash
# 主要分支
main          # 生產環境
develop       # 開發環境 
feature/*     # 功能開發分支
hotfix/*      # 緊急修復分支

# 標籤策略
v3.0.0        # 主版本
v3.0.1        # 修正版本
v3.1.0        # 功能版本
```

#### 部署策略
```yaml
部署流程:
1. 功能開發 → feature 分支
2. 測試完成 → 合併到 develop
3. 穩定驗證 → 合併到 main
4. 自動部署 → Cloud Run 生產環境

回退策略:
1. 保留前一版本映像
2. 一鍵回退功能
3. 資料庫兼容性檢查
```

---

## 開發環境設定

### 本地開發環境

#### 必要軟體安裝
```bash
# Python 環境
Python 3.11+
pip (最新版)

# Google Cloud SDK
curl https://sdk.cloud.google.com | bash
gcloud init
gcloud auth application-default login

# Docker (可選，用於本地容器測試)
Docker Desktop
```

#### 專案設定
```bash
# 1. 克隆專案
git clone https://github.com/matt0taiwan/sibyl-ai-linebot
cd sibyl-ai-linebot

# 2. 建立虛擬環境
python -m venv venv
source venv/bin/activate  # Linux/Mac
# 或 venv\Scripts\activate  # Windows

# 3. 安裝依賴
pip install -r requirements.txt

# 4. 設定環境變數
cp .env.example .env
# 編輯 .env 填入必要的金鑰
```

#### .env 範例檔案
```bash
# .env.example
# LINE Bot 設定
LINE_CHANNEL_ACCESS_TOKEN=your_line_channel_access_token
LINE_CHANNEL_SECRET=your_line_channel_secret

# OpenAI 設定
OPENAI_API_KEY=your_openai_api_key

# Google Sheets 設定
SHEET_ID=1_4R_ftouDHhUXygW3ab7YlEzLWbXytwudfOegUIhu5Y

# AI 模型設定
MODEL_NAME=gpt-4o
MODEL_EMBED=text-embedding-3-large
MODEL_TEMPERATURE=0.85

# 檢索參數
CONTEXT_K=5
SIM_THRESHOLD=0.50
VECTOR_SIM_THRESHOLD=0.60
WEIGHT_PRIORITY=0.10

# 快取設定
FAQ_CACHE_TTL=3600

# 開發模式設定
DEBUG=true
LOG_LEVEL=INFO
```

### 本地測試工具

#### ngrok 設定 (用於 LINE Webhook 測試)
```bash
# 安裝 ngrok
brew install ngrok  # Mac
# 或下載 https://ngrok.com/download

# 啟動本地伺服器
uvicorn app.main:app --host 0.0.0.0 --port 8080 --reload

# 另一個終端開啟 tunnel
ngrok http 8080

# 將 ngrok URL 設定到 LINE Developer Console
# 例如: https://abc123.ngrok.io/webhook
```

#### 單元測試設定
```bash
# 安裝測試依賴
pip install pytest pytest-asyncio pytest-mock

# 執行測試
pytest tests/ -v

# 測試覆蓋率
pip install pytest-cov
pytest tests/ --cov=app --cov-report=html
```

### IDE 配置建議

#### VSCode 設定
```json
// .vscode/settings.json
{
    "python.defaultInterpreterPath": "./venv/bin/python",
    "python.linting.enabled": true,
    "python.linting.flake8Enabled": true,
    "python.formatting.provider": "black",
    "python.sortImports.args": ["--profile", "black"],
    "[python]": {
        "editor.formatOnSave": true,
        "editor.codeActionsOnSave": {
            "source.organizeImports": true
        }
    }
}
```

#### 推薦擴充套件
```json
// .vscode/extensions.json
{
    "recommendations": [
        "ms-python.python",
        "ms-python.flake8",
        "ms-python.black-formatter",
        "ms-python.isort",
        "bradlc.vscode-tailwindcss",
        "ms-vscode.vscode-json"
    ]
}
```

---

## 程式碼規範

### Python 編碼標準

#### PEP 8 合規性
```python
# 行長度限制
最大行長度: 88 字元 (Black formatter 預設)

# 命名規範
函數名稱: snake_case
類別名稱: PascalCase
常數名稱: UPPER_SNAKE_CASE
變數名稱: snake_case
私有方法: _leading_underscore

# 範例
class MessageProcessor:
    """訊息處理器類別"""
    
    MAX_RETRY_COUNT = 3
    
    def __init__(self):
        self.retry_count = 0
        self._private_method()
    
    def process_message(self, user_message: str) -> str:
        """處理用戶訊息"""
        return self._generate_response(user_message)
    
    def _generate_response(self, message: str) -> str:
        """私有方法：生成回應"""
        pass
```

#### 類型提示規範
```python
from typing import Dict, List, Optional, Union, Tuple, Any
from dataclasses import dataclass
from datetime import datetime

# 函數類型提示
def process_intent(
    user_id: str,
    message: str,
    context: Optional[Dict[str, Any]] = None
) -> Tuple[str, float]:
    """
    處理用戶意圖
    
    Args:
        user_id: 用戶唯一識別碼
        message: 用戶訊息內容
        context: 可選的上下文資訊
        
    Returns:
        Tuple[str, float]: (識別的意圖, 信心度分數)
    """
    pass

# 資料類別定義
@dataclass
class ConversationState:
    user_id: str
    current_step: int
    collected_data: Dict[str, Any]
    started_at: datetime
    expires_at: datetime
```

#### 文檔字串標準
```python
def calculate_similarity(text1: str, text2: str) -> float:
    """
    計算兩個文字之間的相似度
    
    使用 Levenshtein 距離算法計算文字相似度，
    返回 0.0 到 1.0 之間的相似度分數。
    
    Args:
        text1 (str): 第一個比較文字
        text2 (str): 第二個比較文字
        
    Returns:
        float: 相似度分數，1.0 表示完全相同，0.0 表示完全不同
        
    Example:
        >>> calculate_similarity("hello", "hello")
        1.0
        >>> calculate_similarity("hello", "world")
        0.2
        
    Raises:
        ValueError: 當輸入為空字串時
    """
    if not text1 or not text2:
        raise ValueError("輸入文字不能為空")
    
    # 實作邏輯...
    pass
```

### 錯誤處理標準

#### 異常處理模式
```python
import logging
from typing import Optional

logger = logging.getLogger(__name__)

class SibylError(Exception):
    """Sibyl AI 基礎異常類別"""
    pass

class OpenAIError(SibylError):
    """OpenAI API 相關錯誤"""
    pass

class FirestoreError(SibylError):
    """Firestore 相關錯誤"""
    pass

# 錯誤處理範例
async def call_openai_api(prompt: str) -> Optional[str]:
    """調用 OpenAI API 並處理錯誤"""
    try:
        response = openai_client.chat.completions.create(
            model=MODEL_NAME,
            messages=[{"role": "user", "content": prompt}],
            temperature=MODEL_TEMPERATURE
        )
        return response.choices[0].message.content
        
    except openai.RateLimitError as e:
        logger.warning(f"OpenAI rate limit exceeded: {e}")
        # 實作指數退避重試
        await asyncio.sleep(2 ** retry_count)
        raise OpenAIError("API 調用頻率限制")
        
    except openai.APIError as e:
        logger.error(f"OpenAI API error: {e}")
        raise OpenAIError(f"API 錯誤: {e}")
        
    except Exception as e:
        logger.error(f"Unexpected error calling OpenAI: {e}", exc_info=True)
        raise SibylError("未預期的錯誤")
```

#### 日誌記錄標準
```python
import logging
from datetime import datetime

# 日誌配置
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('sibyl.log')
    ]
)

logger = logging.getLogger(__name__)

# 日誌使用範例
def process_user_message(user_id: str, message: str):
    """處理用戶訊息並記錄日誌"""
    logger.info(f"Processing message from user {user_id}")
    
    try:
        # 處理邏輯
        result = handle_message(message)
        logger.info(f"Successfully processed message for user {user_id}")
        return result
        
    except Exception as e:
        logger.error(f"Error processing message for user {user_id}: {e}", exc_info=True)
        raise

# 效能監控日誌
def log_performance(func_name: str, duration_ms: int, user_id: str = None):
    """記錄效能指標"""
    logger.info(f"Performance: {func_name} took {duration_ms}ms", extra={
        'function': func_name,
        'duration_ms': duration_ms,
        'user_id': user_id,
        'timestamp': datetime.now().isoformat()
    })
```

### 測試標準

#### 單元測試結構
```python
# tests/test_intent_classifier.py
import pytest
import asyncio
from unittest.mock import Mock, patch
from app.intent_classifier import LLMIntentClassifier, IntentType

class TestLLMIntentClassifier:
    """意圖分類器測試"""
    
    @pytest.fixture
    def classifier(self):
        """測試用分類器實例"""
        return LLMIntentClassifier()
    
    @pytest.mark.asyncio
    async def test_price_inquiry_classification(self, classifier):
        """測試價格詢問意圖識別"""
        # Arrange
        user_message = "塔位價格多少錢？"
        
        # Mock OpenAI response
        mock_response = {
            "intent": "price_inquiry",
            "confidence": 0.95,
            "extracted_data": {},
            "need_human": False,
            "reason": "明確的價格詢問"
        }
        
        with patch('app.config.openai_client.chat.completions.create') as mock_openai:
            mock_openai.return_value.choices[0].message.content = json.dumps(mock_response)
            
            # Act
            result = await classifier.classify(user_message)
            
            # Assert
            assert result.primary_intent == IntentType.PRICE_INQUIRY
            assert result.confidence == 0.95
            assert not result.need_human
    
    @pytest.mark.asyncio
    async def test_openai_api_failure(self, classifier):
        """測試 OpenAI API 失敗處理"""
        with patch('app.config.openai_client.chat.completions.create') as mock_openai:
            mock_openai.side_effect = Exception("API Error")
            
            result = await classifier.classify("測試訊息")
            
            # 應該回退到一般諮詢
            assert result.primary_intent == IntentType.GENERAL_INQUIRY
            assert result.need_human == True
```

#### 整合測試範例
```python
# tests/test_integration.py
import pytest
from fastapi.testclient import TestClient
from app.main import app

class TestWebhookIntegration:
    """Webhook 整合測試"""
    
    @pytest.fixture
    def client(self):
        return TestClient(app)
    
    def test_health_check(self, client):
        """測試健康檢查端點"""
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["status"] in ["healthy", "degraded"]
    
    def test_webhook_signature_validation(self, client):
        """測試 Webhook 簽名驗證"""
        # 無效簽名應該被拒絕
        response = client.post("/webhook", 
                             headers={"X-Line-Signature": "invalid"},
                             json={"events": []})
        assert response.status_code == 403
```

---

## 運維監控

### 監控指標設計

#### 系統健康指標
```python
# app/monitoring.py
from prometheus_client import Counter, Histogram, Gauge
import time
import functools

# 指標定義
REQUEST_COUNT = Counter('sibyl_requests_total', 'Total requests', ['method', 'endpoint'])
REQUEST_DURATION = Histogram('sibyl_request_duration_seconds', 'Request duration')
ACTIVE_CONVERSATIONS = Gauge('sibyl_active_conversations', 'Active conversations')
OPENAI_API_CALLS = Counter('sibyl_openai_calls_total', 'OpenAI API calls', ['model'])
OPENAI_TOKEN_USAGE = Counter('sibyl_openai_tokens_total', 'OpenAI tokens used', ['type'])

# 監控裝飾器
def monitor_performance(func):
    """監控函數執行效能"""
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        start_time = time.time()
        try:
            result = await func(*args, **kwargs)
            return result
        finally:
            duration = time.time() - start_time
            REQUEST_DURATION.observe(duration)
            
            # 記錄效能日誌
            if duration > 5.0:  # 超過 5 秒記錄警告
                logger.warning(f"Slow function {func.__name__}: {duration:.2f}s")
    
    return wrapper

# 使用範例
@monitor_performance
async def process_message(user_id: str, message: str):
    """被監控的訊息處理函數"""
    REQUEST_COUNT.labels(method='process', endpoint='message').inc()
    # 處理邏輯...
```

#### 業務指標監控
```python
# 業務指標收集
class BusinessMetrics:
    def __init__(self):
        self.intent_accuracy = Histogram('sibyl_intent_accuracy', 'Intent classification accuracy')
        self.conversation_completion = Counter('sibyl_conversations_completed', 'Completed conversations')
        self.human_handover = Counter('sibyl_human_handover', 'Human handover events', ['reason'])
        self.faq_hit_rate = Histogram('sibyl_faq_hit_rate', 'FAQ hit rate')
    
    def record_intent_classification(self, confidence: float):
        """記錄意圖識別準確度"""
        self.intent_accuracy.observe(confidence)
    
    def record_conversation_completion(self, intent_type: str):
        """記錄對話完成"""
        self.conversation_completion.labels(intent=intent_type).inc()
    
    def record_human_handover(self, reason: str):
        """記錄人工接管"""
        self.human_handover.labels(reason=reason).inc()

business_metrics = BusinessMetrics()
```

### 日誌聚合與分析

#### 結構化日誌
```python
import json
import logging
from datetime import datetime

class JSONFormatter(logging.Formatter):
    """JSON 格式日誌"""
    
    def format(self, record):
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        
        # 添加額外字段
        if hasattr(record, 'user_id'):
            log_entry['user_id'] = record.user_id
        if hasattr(record, 'request_id'):
            log_entry['request_id'] = record.request_id
        if hasattr(record, 'duration_ms'):
            log_entry['duration_ms'] = record.duration_ms
            
        return json.dumps(log_entry, ensure_ascii=False)

# 配置結構化日誌
def setup_logging():
    """設定結構化日誌"""
    logger = logging.getLogger()
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)

# 使用範例
logger = logging.getLogger(__name__)

def log_user_interaction(user_id: str, action: str, duration_ms: int):
    """記錄用戶互動"""
    logger.info(f"User interaction: {action}", extra={
        'user_id': user_id,
        'action': action,
        'duration_ms': duration_ms
    })
```

#### Cloud Logging 整合
```python
# Google Cloud Logging 設定
from google.cloud import logging as cloud_logging

def setup_cloud_logging():
    """設定 Google Cloud Logging"""
    client = cloud_logging.Client()
    client.setup_logging()
    
    # 自訂日誌處理
    cloud_handler = cloud_logging.handlers.CloudLoggingHandler(client)
    cloud_handler.setFormatter(JSONFormatter())
    
    logger = logging.getLogger()
    logger.addHandler(cloud_handler)
```

### 告警與通知

#### 告警規則定義
```python
# app/alerting.py
import smtplib
from email.mime.text import MIMEText
from datetime import datetime, timedelta

class AlertManager:
    def __init__(self):
        self.alert_thresholds = {
            'error_rate': 0.05,        # 5% 錯誤率
            'response_time': 10.0,     # 10秒回應時間
            'memory_usage': 0.90,      # 90% 記憶體使用率
            'openai_failures': 5       # 5次 OpenAI 失敗
        }
        self.alert_intervals = {}  # 防止重複告警
    
    def check_error_rate(self, error_count: int, total_count: int):
        """檢查錯誤率"""
        if total_count == 0:
            return
            
        error_rate = error_count / total_count
        if error_rate > self.alert_thresholds['error_rate']:
            self.send_alert(
                'High Error Rate',
                f'Error rate: {error_rate:.2%} (threshold: {self.alert_thresholds["error_rate"]:.2%})'
            )
    
    def check_response_time(self, avg_response_time: float):
        """檢查回應時間"""
        if avg_response_time > self.alert_thresholds['response_time']:
            self.send_alert(
                'Slow Response Time',
                f'Average response time: {avg_response_time:.2f}s'
            )
    
    def send_alert(self, title: str, message: str):
        """發送告警通知"""
        # 檢查告警間隔，避免重複發送
        alert_key = f"{title}_{datetime.now().strftime('%Y%m%d%H')}"  # 每小時最多一次
        
        if alert_key in self.alert_intervals:
            return
            
        self.alert_intervals[alert_key] = datetime.now()
        
        # 發送告警 (可以是 email, Slack, 等)
        self._send_email_alert(title, message)
        logger.error(f"ALERT: {title} - {message}")
    
    def _send_email_alert(self, title: str, message: str):
        """發送 email 告警"""
        # 實作 email 發送邏輯
        pass

alert_manager = AlertManager()
```

### 效能監控儀表板

#### Grafana Dashboard 配置
```json
{
  "dashboard": {
    "title": "Sibyl AI Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(sibyl_requests_total[5m])",
            "legendFormat": "{{method}} {{endpoint}}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(sibyl_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "

## 最新更新 (2025-07-18)

### 防濫用機制實作
- **AI 使用量限制**：每用戶每日 100 次 AI 對話限制
- **時區處理**：使用台灣時區（UTC+8）計算每日限制，午夜重置
- **資料儲存**：使用 Firestore 集合 `user_ai_usage` 儲存使用記錄
- **自動清理**：超過 7 天的使用記錄自動刪除
- **錯誤處理**：如檢查失敗預設允許使用，避免影響服務
- **交易支援**：使用 Firestore transaction 確保計數準確性

### 實作細節
- **memory_manager.py**：
  - `check_ai_usage_limit()` - 檢查用戶今日使用量
  - `increment_ai_usage()` - 增加使用次數
  - `_get_today_key()` - 取得台灣時區日期
  - `_cleanup_old_usage()` - 清理舊記錄
- **message_processor.py**：
  - 在 AI 模式處理前檢查限制
  - 達到限制時顯示友善提示
- **客服資訊更新**：
  - 專線：04-25376688
  - 時間：09:00-17:00

## 之前更新 (2025-07-17)

### 業務邏輯優化 - 條件性欄位實作
- **動態欄位收集**：根據用戶選擇動態調整需要收集的欄位
- **條件性欄位架構**：
  - 法會報名（拜桌）：新增處理方式選項（代捐/自取/託運[隱藏]）
  - 佛事登記（對年）：新增三選項（單純誦經/請香火回家/塔上合爐）
  - 託運選項：不主動顯示，但當用戶提到託運相關關鍵字時自動識別並處理
- **託運資訊收集**：當選擇託運時，動態新增收件人、收件人電話、收件地址欄位

### 智慧資料擷取強化
- **SmartDataExtractor 擴充**：提取條件性欄位（拜桌處理方式、對年處理方式）
- **欄位預填充**：從單一訊息中提取多個欄位，減少對話輪次
- **同上功能**：託運資訊支援「同上」快速填寫，自動使用報名資訊

### 即時欄位驗證系統
- **新模組 field_validator.py**：
  - 電話號碼格式驗證（手機、市話）
  - 日期時間格式驗證（支援多種格式）
  - 地址完整性驗證
  - 選項有效性驗證（拜桌處理、對年處理）
- **錯誤提示友善化**：提供具體的格式範例和修正建議
- **隱藏選項處理**：託運選項的智慧識別（不顯示但可識別）

### 流程進度視覺化
- **新模組 progress_formatter.py**：
  - 動態進度條顯示（根據條件性欄位調整總數）
  - 已填/待填欄位清單
  - 錯誤時顯示進度上下文
  - 完成摘要格式化（託運資訊顯示在最後）
- **視覺元素**：使用 emoji 增強可讀性（✅ 已完成、👉 當前、⏳ 待填寫）

### 程式碼架構優化
- **ConversationState 增強**：
  - 新增 `conditional_fields` 結構定義條件邏輯
  - 實作 `get_dynamic_fields()` 動態計算欄位列表
  - 實作 `get_next_field()` 智慧取得下一個欄位
- **FlowHandler 整合**：無縫整合驗證器和進度格式化器
- **狀態管理改進**：支援複雜的多層條件欄位邏輯

## 之前更新 (2025-07-16)

### AI 對話智慧提升
- **意圖糾正處理**：修復用戶否定意圖（如「我沒有要預約」）後 AI 停止回應的問題
- **二次確認機制**：實作意圖確認流程，低信心度或模糊訊息會先確認再開始
- **待確認狀態**：新增 `pending_confirmation` 狀態管理
- **時間性描述**：改進「對年快到了」等時間性表達的意圖識別

### Template Message 修復與優化
- **發送問題修復**：解決 TemplateMessage 類型導入問題
- **URL 驗證強化**：自動將 HTTP 轉換為 HTTPS（LINE 要求）
- **錯誤處理改進**：增加詳細日誌和備援機制
- **降級策略**：Template 建立失敗時自動使用一般圖片訊息

### 價格管理系統革新
- **FAQ 化實現**：價格詢問從硬編碼改為 FAQ 系統（ID: P001）
- **業務策略保留**：維持「先價值後價格」的銷售話術
- **動態管理**：業務團隊可在 Google Sheets 自行更新
- **管理文檔**：建立完整的價格 FAQ 管理指南

### 程式碼結構優化
- **FlowHandler 增強**：新增 `_is_ambiguous_message()` 和 `_confirm_intent_before_flow()`
- **ConversationState 擴充**：加入 `pending_intent` 和 `pending_data` 欄位
- **錯誤處理統一**：所有 Template Message 錯誤都有詳細記錄

## 之前更新 (2025-07-15)

### 雙模式系統完善
- **傳統模式**：精確關鍵字匹配，支援 LINE OA 風格自動回覆
- **AI 模式**：智慧對話，永久保持（移除 30 分鐘超時）
- **模式切換**：流暢切換，狀態持久化於 Firestore

### AI 模式切換增強
- "0" 關鍵字現在可從 Google Sheets 控制文案內容
- 系統先檢查 Sheets 自動回覆，再執行模式切換
- 其他 AI 切換關鍵字保持預設歡迎訊息
