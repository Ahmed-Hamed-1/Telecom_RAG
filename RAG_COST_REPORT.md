# Telecom RAG - Final Assignment Cost Report

## 1. Docker Commands
Since Docker was utilized for containerization, here are the actual commands prepared and validated for this environment:

**Build Command:**
```powershell
docker build -t telecom-rag .
```

**Run Command:**
```powershell
docker run -d -p 8000:8000 --env-file .env telecom-rag
```
*(Note: `.env` is deliberately excluded from `.dockerignore` for this test run only to allow runtime variables, but normally `.env` is loaded via `--env-file` so it is not baked into the image layer itself.)*

## 2. Container Health Check Result
**Request:** `GET http://localhost:8000/`
**Response:**
```json
{
  "status": "healthy",
  "app_name": "Telecom RAG API",
  "version": "1.0.0",
  "docs_url": "/docs"
}
```
**Swagger URL:** `http://localhost:8000/docs`

## 3. RAG Request
**Containerized endpoint:** `POST /api/v1/query`
**HTTP method:** `POST`
**Full request JSON:**
```json
{
  "ticket": "النت فاصل عندي ولمبة DSL بتنور وتطفي"
}
```

## 4. RAG Response
**Actual HTTP status code:** `200 OK`
**Actual response JSON:**
```json
{
  "ticket": "النت فاصل عندي ولمبة DSL بتنور وتطفي",
  "response": "أهلاً بحضرتك يا فندم، بعتذر لحضرتك جداً عن المشكلة دي ومقّدر إزعاج انقطاع الخدمة، وأنا مع حضرتك من خدمة العملاء وهساعدك نحلها فوراً.\n\nلمبة الـ DSL لما بتكون بتنور وتطفي، ده معناه إن فيه عدم قدرة على التزامن مع السنترال.\n\nعشان نمشي في خطوات الحل طبقا للتعليمات المعتمدة:\n1. **الخطوة الأولى:** استأذن حضرتك تقفل جهاز الراوتر وتعيش تشغيله تاني (تعمل إعادة تشغيل) وتستنى دقيقتين، وتتأكد تماماً من السلك الداخلي والتوصيلات إنها متوصلة كويس ومش مهزوزة.\n\n2. **الخطوات التالية حسب الحالة:**\n   - لو المشكلة مستمرة بعد إعادة التشغيل وفحص السلك لأكتر من 24 ساعة، هقوم بتصعيد المشكلة فوراً لفريق السنترال المختص.\n   - لو لمبة الـ DSL مستمرة بتنور وتطفي (أو فاصلة تماماً) بقالها 3 أيام متتالية، أو لو كان هامش ضوضاء الخط (Line noise margin) أقل من 6dB، فبناءً على قواعد الخدمة هيتم إرسال فني صيانة لمقر حضرتك لفحص الخط ميدانياً.\n\nيا ريت توضح لي هل جربت إعادة تشغيل الراوتر وفحص التوصيلات؟ وبقالها كام يوم المشكلة دي قائمة عند حضرتك بالضبط عشان أبدأ في الإجراء المناسب لحضرتك فوراً؟",
  "sources_count": 20,
  "execution_time_seconds": 21.69,
  "prompt_tokens": 2507,
  "completion_tokens": 1852,
  "total_tokens": 4359
}
```

## 5. Model & Token Usage
- **Actual Gemini model used:** `gemini-3.6-flash`
- **Input / Prompt Tokens:** 2507
- **Output / Completion Tokens:** 1852
- **Total Tokens:** 4359

## 6. Request Cost Calculation
**Pricing Source:** Official introductory API Pricing for Gemini 3.6 Flash (valid through December 31, 2026).
- **Input token price:** $0.75 per 1,000,000 tokens
- **Output token price:** $3.75 per 1,000,000 tokens

**Cost Formula & Actual Calculated Cost:**
- **Input cost:** 2507 / 1,000,000 × $0.75 = **$0.00188025**
- **Output cost:** 1852 / 1,000,000 × $3.75 = **$0.00694500**
- **Total Cost:** $0.00188025 + $0.00694500 = **$0.00882525**

**Explanation of the calculation:**
The API processes the incoming prompt (including the 20 retrieved FAISS chunks acting as context, the system instructions, and the user's ticket) for a total of 2507 input tokens. It generated 1852 output tokens to form the detailed Arabic response. We multiply these counts by the per-token rate ($0.75/1M and $3.75/1M respectively) to find the exact fraction of a cent this query cost.

**Assumptions or limitations:**
- Pricing does not include other potential GCP infrastructure costs (like network egress or Vertex AI platform fees if accessed via Vertex).
- This calculation assumes a standard success response (HTTP 200).
- The FAISS index is stored locally, so there are no recurring vector database cloud fees (e.g., Pinecone or Weaviate costs) included in this transaction.
