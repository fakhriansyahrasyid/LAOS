{
  "name": "LAOS Finance - Main Workflow (v1.1)",
  "nodes": [
    {
      "parameters": { "updates": ["message"] },
      "id": "d19d656c-60a0-473d-bd28-2f93fcca51c2",
      "name": "01 - Telegram Trigger",
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1,
      "position": [-1700, 300],
      "webhookId": "laos-finance-telegram-webhook"
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "chat_id", "value": "={{$json[\"message\"][\"chat\"][\"id\"]}}" },
            { "name": "user_id", "value": "={{$json[\"message\"][\"from\"][\"id\"]}}" },
            { "name": "username", "value": "={{$json[\"message\"][\"from\"][\"username\"]}}" },
            { "name": "message_text", "value": "={{$json[\"message\"][\"text\"]}}" }
          ]
        },
        "options": {}
      },
      "id": "43efc9ef-9579-42e5-a852-fe2299585693",
      "name": "02 - Extract Message",
      "type": "n8n-nodes-base.set",
      "typeVersion": 1,
      "position": [-1460, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "={{$env[\"LAOS_CORE_URL\"]}}/api/identity/validate",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "user_id", "value": "={{$json[\"user_id\"]}}" },
            { "name": "channel", "value": "telegram" }
          ]
        },
        "options": {}
      },
      "id": "88bda33c-237d-4c4f-8a91-5111ae2e50b5",
      "name": "03 - Identity Service",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [-1220, 300]
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            { "value1": "={{$json[\"valid\"]}}", "value2": true }
          ]
        }
      },
      "id": "836006af-776c-415b-ac3b-848c9c603ffc",
      "name": "04 - IF Identity Valid",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [-980, 300]
    },
    {
      "parameters": {
        "chatId": "={{$node[\"02 - Extract Message\"].json[\"chat_id\"]}}",
        "text": "Maaf, sesi Anda tidak valid atau tidak memiliki akses. Silakan hubungi admin LAOS.",
        "additionalFields": {}
      },
      "id": "356b6317-7fd2-415d-94d0-01a9fdfb120a",
      "name": "05 - Unauthorized Response",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [-720, 480]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "={{$env[\"LAOS_CORE_URL\"]}}/api/memory/query",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "user_id", "value": "={{$node[\"02 - Extract Message\"].json[\"user_id\"]}}" },
            { "name": "scope", "value": "finance" },
            { "name": "query", "value": "={{$node[\"02 - Extract Message\"].json[\"message_text\"]}}" }
          ]
        },
        "options": {}
      },
      "id": "a5851dc1-20e4-4aa3-850d-c8eeaa226633",
      "name": "06 - Memory Retrieval",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [-720, 200],
      "notes": "Mengambil context dari Notion (SOP/KPI/OKR/Policy) dan ERPNext, sesuai Memory Mapping TDD bab 7"
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"model\": \"gpt-4o\",\n  \"temperature\": 0.1,\n  \"messages\": [\n    {\"role\": \"system\", \"content\": \"Anda adalah Intent Classifier untuk LAOS Finance. Klasifikasikan permintaan ke salah satu intent: get_cash, get_journal, create_journal, get_invoice, get_payment, get_gl, get_account, get_report_pl, unknown. Balas HANYA JSON: {\\\"intent\\\": \\\"...\\\", \\\"parameters\\\": {}, \\\"requires_write\\\": false}\"},\n    {\"role\": \"user\", \"content\": {{JSON.stringify($node[\"02 - Extract Message\"].json[\"message_text\"] + \" | Context: \" + JSON.stringify($json))}} }\n  ]\n}",
        "options": {}
      },
      "id": "a1f563d6-49c6-44d1-8f8b-6f69badb7e21",
      "name": "07 - Intent Detection (OpenAI HTTP)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [-480, 300],
      "notes": "Auth: set Header Auth credential (Authorization: Bearer <OPENAI_API_KEY>) manual di UI n8n setelah import"
    },
    {
      "parameters": {
        "jsCode": "// Parse hasil OpenAI & tentukan endpoint ERPNext berdasarkan intent\nconst body = $input.first().json;\nlet raw = '{}';\ntry {\n  raw = body.choices[0].message.content;\n} catch (e) {\n  raw = '{}';\n}\nlet parsed;\ntry {\n  parsed = JSON.parse(raw);\n} catch (e) {\n  parsed = { intent: 'unknown', parameters: {}, requires_write: false };\n}\n\nconst erpBase = $env.ERPNEXT_URL || '';\nconst map = {\n  get_cash:        { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_cash' },\n  get_journal:      { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_journal' },\n  create_journal:   { method: 'POST', url: erpBase + '/api/resource/Journal Entry' },\n  get_invoice:      { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_invoice' },\n  get_payment:      { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_payment' },\n  get_gl:           { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_gl' },\n  get_account:      { method: 'GET',  url: erpBase + '/api/method/laos.finance.get_account' },\n  get_report_pl:    { method: 'GET',  url: erpBase + '/api/finance/report/pl' }\n};\n\nconst route = map[parsed.intent] || null;\n\nreturn [{\n  json: {\n    intent: parsed.intent || 'unknown',\n    parameters: parsed.parameters || {},\n    requires_write: !!parsed.requires_write,\n    erp_method: route ? route.method : null,\n    erp_url: route ? route.url : null,\n    is_known_intent: !!route\n  }\n}];"
      },
      "id": "17496510-5c0e-4c49-a726-46f4f00aa30c",
      "name": "08 - Parse Intent & Build ERP Route",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [-240, 300]
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            { "value1": "={{$json[\"is_known_intent\"]}}", "value2": true }
          ]
        }
      },
      "id": "9c65d631-10ba-45e0-9ef6-a683d8dc550b",
      "name": "09 - IF Intent Dikenali",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [0, 300]
    },
    {
      "parameters": {
        "chatId": "={{$node[\"02 - Extract Message\"].json[\"chat_id\"]}}",
        "text": "Maaf, saya belum bisa memahami permintaan Anda. Coba contoh: 'tampilkan laporan laba rugi bulan ini' atau 'cek saldo kas'.",
        "additionalFields": {}
      },
      "id": "0bae456f-691c-48f8-b5b2-5166c8990195",
      "name": "09b - Unknown Intent Response",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [240, 480]
    },
    {
      "parameters": {
        "method": "={{$json[\"erp_method\"]}}",
        "url": "={{$json[\"erp_url\"]}}",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{JSON.stringify($json[\"parameters\"])}}",
        "options": {}
      },
      "id": "ac9cb4ce-504b-49d3-84fb-103aed06f1b9",
      "name": "10 - ERP Connector (Generic)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [240, 220],
      "notes": "Auth: set credential ke ERPNext (API Key/Secret) manual di UI n8n setelah import. Method & URL diambil dinamis dari node 08."
    },
    {
      "parameters": {
        "method": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            { "name": "Content-Type", "value": "application/json" }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"model\": \"gpt-4o\",\n  \"temperature\": 0.3,\n  \"messages\": [\n    {\"role\": \"system\", \"content\": \"Anda adalah AI Analyst LAOS Finance. Analisis data keuangan berikut, buat ringkasan singkat, insight utama, dan rekomendasi bila relevan. Bahasa Indonesia, profesional, ringkas.\"},\n    {\"role\": \"user\", \"content\": {{JSON.stringify(\"Intent: \" + $node[\"08 - Parse Intent & Build ERP Route\"].json[\"intent\"] + \" | Data ERP: \" + JSON.stringify($json))}} }\n  ]\n}",
        "options": {}
      },
      "id": "1baf0547-7344-45cb-9355-bfc3a4fd9fac",
      "name": "11 - AI Analysis (OpenAI HTTP)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [480, 220],
      "notes": "Auth: set Header Auth credential (Authorization: Bearer <OPENAI_API_KEY>) manual di UI n8n setelah import"
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            { "value1": "={{$node[\"08 - Parse Intent & Build ERP Route\"].json[\"requires_write\"]}}", "value2": true }
          ]
        }
      },
      "id": "69b132a0-c9a5-45ae-a060-ecaa3cc79672",
      "name": "12 - IF Approval Needed",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [720, 220]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "={{$env[\"LAOS_CORE_URL\"]}}/api/approval/request",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "user_id", "value": "={{$node[\"02 - Extract Message\"].json[\"user_id\"]}}" },
            { "name": "intent", "value": "={{$node[\"08 - Parse Intent & Build ERP Route\"].json[\"intent\"]}}" },
            { "name": "payload", "value": "={{JSON.stringify($node[\"08 - Parse Intent & Build ERP Route\"].json[\"parameters\"])}}" }
          ]
        },
        "options": {}
      },
      "id": "05aee3b1-54b4-41fe-b4f4-ae4599bcbf96",
      "name": "13 - Approval Engine: Request",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [960, 120],
      "notes": "SLA Approval <= 10 detik sesuai TDD bab 12. Respons endpoint ini diasumsikan berisi field 'decision'."
    },
    {
      "parameters": { "amount": 10, "unit": "seconds" },
      "id": "72bff96d-4e19-45e0-926c-1671929b40fd",
      "name": "13b - Wait for Decision",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1,
      "position": [1200, 120]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            { "value1": "={{$json[\"decision\"]}}", "value2": "approved" }
          ]
        }
      },
      "id": "8b74cc56-2d6d-4bd1-9231-4b62455d3ea5",
      "name": "14 - IF Approved",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [1440, 120]
    },
    {
      "parameters": {
        "chatId": "={{$node[\"02 - Extract Message\"].json[\"chat_id\"]}}",
        "text": "={{\"Permintaan Anda ditolak/dibatalkan oleh approver. Intent: \" + $node[\"08 - Parse Intent & Build ERP Route\"].json[\"intent\"]}}",
        "additionalFields": {}
      },
      "id": "b9b606e3-5c9f-48bf-99fa-542c43c76723",
      "name": "14b - Rejected Response",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [1680, 280]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "={{$env[\"LAOS_CORE_URL\"]}}/api/audit/log",
        "sendBody": true,
        "bodyParameters": {
          "parameters": [
            { "name": "user_id", "value": "={{$node[\"02 - Extract Message\"].json[\"user_id\"]}}" },
            { "name": "intent", "value": "={{$node[\"08 - Parse Intent & Build ERP Route\"].json[\"intent\"]}}" },
            { "name": "ai_prompt", "value": "={{$node[\"02 - Extract Message\"].json[\"message_text\"]}}" },
            { "name": "approval_status", "value": "={{$json[\"decision\"] || \"not_required\"}}" }
          ]
        },
        "options": {}
      },
      "id": "a0dfdc7c-1783-4052-b3ce-594701a46fc5",
      "name": "15 - Audit Log (LAOS Core API)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.2,
      "position": [1680, 20],
      "notes": "Diganti dari koneksi Postgres langsung menjadi panggilan ke LAOS Core Audit API supaya skema DB tetap terenkapsulasi di Core (bab 6 & 10 TDD). Jika ingin tulis langsung ke Postgres, tambahkan node Postgres secara manual di n8n UI."
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "final_message", "value": "={{$node[\"11 - AI Analysis (OpenAI HTTP)\"].json[\"choices\"][0][\"message\"][\"content\"]}}" }
          ]
        },
        "options": {}
      },
      "id": "b6a0f6a5-bfc3-480a-9df3-2fa921b4d0fb",
      "name": "16 - Format Response",
      "type": "n8n-nodes-base.set",
      "typeVersion": 1,
      "position": [1920, 20]
    },
    {
      "parameters": {
        "chatId": "={{$node[\"02 - Extract Message\"].json[\"chat_id\"]}}",
        "text": "={{$json[\"final_message\"]}}",
        "additionalFields": {}
      },
      "id": "224386b1-c17a-4efc-b960-c18eb5d1d371",
      "name": "17 - Send Telegram Response",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [2160, 20]
    }
  ],
  "connections": {
    "01 - Telegram Trigger": { "main": [[{ "node": "02 - Extract Message", "type": "main", "index": 0 }]] },
    "02 - Extract Message": { "main": [[{ "node": "03 - Identity Service", "type": "main", "index": 0 }]] },
    "03 - Identity Service": { "main": [[{ "node": "04 - IF Identity Valid", "type": "main", "index": 0 }]] },
    "04 - IF Identity Valid": {
      "main": [
        [{ "node": "06 - Memory Retrieval", "type": "main", "index": 0 }],
        [{ "node": "05 - Unauthorized Response", "type": "main", "index": 0 }]
      ]
    },
    "06 - Memory Retrieval": { "main": [[{ "node": "07 - Intent Detection (OpenAI HTTP)", "type": "main", "index": 0 }]] },
    "07 - Intent Detection (OpenAI HTTP)": { "main": [[{ "node": "08 - Parse Intent & Build ERP Route", "type": "main", "index": 0 }]] },
    "08 - Parse Intent & Build ERP Route": { "main": [[{ "node": "09 - IF Intent Dikenali", "type": "main", "index": 0 }]] },
    "09 - IF Intent Dikenali": {
      "main": [
        [{ "node": "10 - ERP Connector (Generic)", "type": "main", "index": 0 }],
        [{ "node": "09b - Unknown Intent Response", "type": "main", "index": 0 }]
      ]
    },
    "10 - ERP Connector (Generic)": { "main": [[{ "node": "11 - AI Analysis (OpenAI HTTP)", "type": "main", "index": 0 }]] },
    "11 - AI Analysis (OpenAI HTTP)": { "main": [[{ "node": "12 - IF Approval Needed", "type": "main", "index": 0 }]] },
    "12 - IF Approval Needed": {
      "main": [
        [{ "node": "13 - Approval Engine: Request", "type": "main", "index": 0 }],
        [{ "node": "15 - Audit Log (LAOS Core API)", "type": "main", "index": 0 }]
      ]
    },
    "13 - Approval Engine: Request": { "main": [[{ "node": "13b - Wait for Decision", "type": "main", "index": 0 }]] },
    "13b - Wait for Decision": { "main": [[{ "node": "14 - IF Approved", "type": "main", "index": 0 }]] },
    "14 - IF Approved": {
      "main": [
        [{ "node": "15 - Audit Log (LAOS Core API)", "type": "main", "index": 0 }],
        [{ "node": "14b - Rejected Response", "type": "main", "index": 0 }]
      ]
    },
    "15 - Audit Log (LAOS Core API)": { "main": [[{ "node": "16 - Format Response", "type": "main", "index": 0 }]] },
    "16 - Format Response": { "main": [[{ "node": "17 - Send Telegram Response", "type": "main", "index": 0 }]] }
  },
  "pinData": {},
  "settings": {
    "executionOrder": "v1"
  },
  "staticData": null
}
