# วิธีการให้ AI นำไปใช้งานได้จริง 🤖

คู่มือเชิงปฏิบัติสำหรับการทำให้ AI skills สามารถใช้งานได้จริงในโปรเจคของคุณ

---

## 📚 สารบัญ

1. [ภาพรวมการใช้งาน](#ภาพรวมการใช้งาน)
2. [วิธีการเชื่อมต่อกับ AI](#วิธีการเชื่อมต่อกับ-ai)
3. [การ Integrate กับ GitHub](#การ-integrate-กับ-github)
4. [Webhook และ Event Handling](#webhook-และ-event-handling)
5. [ตัวอย่างจริง](#ตัวอย่างจริง)
6. [Automation Workflow](#automation-workflow)
7. [การตั้งค่า CI/CD](#การตั้งค่า-cicd)
8. [การใช้งาน API](#การใช้งาน-api)

---

## ภาพรวมการใช้งาน

### วิธีการ 3 แบบที่ AI สามารถใช้งานได้:

```
┌──────────────────────────────────────────────────────────────┐
│                   วิธีการใช้งาน AI                          │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ 1️⃣  MANUAL MODE (ใช้เมื่อต้องการ)                        │
│    └─ คลิกปุ่มหรือรันคำสั่ง → AI ทำงาน                   │
│    └─ ใช้: npm run analyze                                 │
│                                                              │
│ 2️⃣  AUTO MODE (อัตโนมัติเมื่อเกิดเหตุการณ์)             │
│    └─ เมื่อ push code → Webhook แจ้ง AI → AI ทำงาน       │
│    └─ เมื่อ PR สร้าง → AI review code                     │
│                                                              │
│ 3️⃣  SCHEDULED MODE (ตามเวลา)                            │
│    └─ ทุกวันเที่ยงคืน → AI scan repo ทั้งหมด              │
│    └─ ทุกสัปดาห์ → AI generate report                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## วิธีการเชื่อมต่อกับ AI

### วิธีที่ 1: API Integration (ใช้ HTTP Request)

#### ขั้นตอน 1: ตั้งค่า API Server

สร้างไฟล์ `ai-server.js`:

```javascript
const express = require('express');
const AISkills = require('./src');
const app = express();

app.use(express.json());

// API endpoint สำหรับวิเคราะห์โค้ด
app.post('/api/analyze', async (req, res) => {
  try {
    const { code, language } = req.body;
    
    const analyzer = new AISkills.LineByLineAnalyzer();
    const result = await analyzer.analyzeEveryStatement(code, {
      language: language || 'javascript'
    });
    
    res.json({
      success: true,
      errors: result.errors,
      warnings: result.warnings,
      qualityScore: result.qualityScore
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// API endpoint สำหรับแก้ไขโค้ด
app.post('/api/fix', async (req, res) => {
  try {
    const { code, language } = req.body;
    
    const fixer = new AISkills.PredictiveErrorFixerAgent({
      autoFix: true,
      iterateUntilClean: true,
      language: language || 'javascript'
    });
    
    const result = await fixer.analyzeAndFixPreemptively(code);
    
    res.json({
      success: true,
      originalCode: code,
      fixedCode: result.fixedCode,
      errorsFound: result.totalErrorsFound,
      errorsFixed: result.totalErrorsFixed,
      iterations: result.iterations,
      report: result.detailedReport
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// API endpoint สำหรับ generate test
app.post('/api/generate-tests', async (req, res) => {
  try {
    const { code, language } = req.body;
    
    const testGen = new AISkills.TestGeneratorAgent();
    const tests = await testGen.generateComprehensiveTests(code, {
      coverage: 80,
      language: language || 'javascript'
    });
    
    res.json({
      success: true,
      tests: tests,
      count: tests.length
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    });
  }
});

// เริ่ม server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`AI Skills API Server running on http://localhost:${PORT}`);
});
```

#### ขั้นตอน 2: รัน Server

```bash
node ai-server.js
# Output: AI Skills API Server running on http://localhost:3000
```

#### ขั้นตอน 3: ใช้งาน API

```bash
# วิเคราะห์โค้ด
curl -X POST http://localhost:3000/api/analyze \
  -H "Content-Type: application/json" \
  -d '{
    "code": "function test() { return user.name; }",
    "language": "javascript"
  }'

# แก้ไขโค้ด
curl -X POST http://localhost:3000/api/fix \
  -H "Content-Type: application/json" \
  -d '{
    "code": "function test() { return user.name; }",
    "language": "javascript"
  }'
```

---

### วิธีที่ 2: GitHub Integration (ใช้ GitHub App)

#### ขั้นตอน 1: สร้าง GitHub App

1. ไปที่ GitHub Settings → Developer settings → GitHub Apps
2. คลิก "New GitHub App"
3. กรอกข้อมูล:

```
App name: AI Code Analyzer
Homepage URL: https://github.com/Nal2awit/ai-github-app-skills
Webhook URL: https://your-server.com/webhook/github
Webhook secret: your-secret-key
```

#### ขั้นตอน 2: ตั้ง Permissions

ให้ GitHub App มีสิทธิ์:
- ✅ Read & Write: Pull requests
- ✅ Read & Write: Issues
- ✅ Read: Repository contents
- ✅ Read: Code

#### ขั้นตอน 3: สร้าง Webhook Handler

สร้างไฟล์ `github-webhook.js`:

```javascript
const express = require('express');
const crypto = require('crypto');
const AISkills = require('./src');

const app = express();
app.use(express.json());

const WEBHOOK_SECRET = process.env.GITHUB_WEBHOOK_SECRET;

// ตรวจสอบลายเซ็น webhook
function verifyWebhook(req, secret) {
  const signature = req.headers['x-hub-signature-256'];
  const body = JSON.stringify(req.body);
  const hash = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');
  
  return crypto.timingSafeEqual(signature, hash);
}

// Handler สำหรับ pull request
app.post('/webhook/github', async (req, res) => {
  try {
    // ตรวจสอบลายเซ็น
    if (!verifyWebhook(req, WEBHOOK_SECRET)) {
      return res.status(401).json({ error: 'Invalid signature' });
    }
    
    const event = req.headers['x-github-event'];
    const payload = req.body;
    
    // เมื่อมี pull request ใหม่
    if (event === 'pull_request' && payload.action === 'opened') {
      console.log(`🔍 Analyzing PR #${payload.pull_request.number}`);
      
      // ได้โค้ดจาก PR
      const prCode = payload.pull_request.body;
      
      // วิเคราะห์
      const analyzer = new AISkills.CodeReviewAgent();
      const review = await analyzer.review(payload.pull_request, {
        strict: true,
        checkBestPractices: true
      });
      
      // ส่ง comment กลับไป GitHub
      // (ใช้ GitHub API เพื่อโพส comment)
      
      res.json({ success: true, message: 'Analysis complete' });
    }
    
    // เมื่อมี push ใหม่
    if (event === 'push') {
      console.log(`📝 Analyzing push to ${payload.ref}`);
      
      // วิเคราะห์โค้ดที่ push
      const fixer = new AISkills.PredictiveErrorFixerAgent({
        autoFix: true,
        iterateUntilClean: true
      });
      
      // ส่งรายงาน
      res.json({ success: true, message: 'Push analysis complete' });
    }
    
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: error.message });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`GitHub Webhook receiver running on port ${PORT}`);
});
```

#### ขั้นตอน 4: Deploy Webhook Server

```bash
# ใช้ ngrok เพื่อ expose local server
ngrok http 3000

# แล้วใช้ ngrok URL ในการ setup GitHub App webhook
```

---

## การ Integrate กับ GitHub

### วิธี 1: GitHub Actions

สร้างไฟล์ `.github/workflows/ai-analyze.yml`:

```yaml
name: AI Code Analysis

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  analyze:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'
    
    - name: Install dependencies
      run: npm install
    
    - name: Run AI Analysis
      run: npm run analyze
      env:
        CI: true
    
    - name: Run AI Auto-Fix
      run: npm run auto-fix
      continue-on-error: true
    
    - name: Run Tests
      run: npm test
    
    - name: Upload Report
      uses: actions/upload-artifact@v2
      with:
        name: analysis-report
        path: ./reports/
    
    - name: Comment PR
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          const report = JSON.parse(fs.readFileSync('./reports/analysis.json', 'utf8'));
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## 🤖 AI Code Analysis Report\n\n${report.summary}`
          });
```

### วิธี 2: Manual Command (รันด้วยมือ)

```bash
# วิเคราะห์ไฟล์เดียว
npm run analyze -- src/main.js

# วิเคราะห์โฟลเดอร์ทั้งหมด
npm run analyze -- src/

# แก้ไขอัตโนมัติ
npm run auto-fix -- src/

# สร้างรายงาน
npm run report -- --output ./reports/
```

---

## Webhook และ Event Handling

### ประเภท Event ที่สนับสนุน

```javascript
// events/handlers.js

const EventHandlers = {
  // เมื่อมี push code
  'push': async (payload) => {
    console.log('📝 Code pushed');
    return await scanPushedCode(payload);
  },
  
  // เมื่อมี PR สร้าง
  'pull_request.opened': async (payload) => {
    console.log('🔍 PR opened, reviewing...');
    return await reviewPullRequest(payload);
  },
  
  // เมื่อ PR ถูกแก้ไข
  'pull_request.synchronize': async (payload) => {
    console.log('♻️  PR updated, re-analyzing...');
    return await reviewPullRequest(payload);
  },
  
  // เมื่อมี Issue สร้าง
  'issues.opened': async (payload) => {
    console.log('⚠️  Issue created');
    return await handleIssue(payload);
  },
  
  // เมื่อมี commit
  'push.commits': async (payload) => {
    console.log('📦 Commits pushed');
    return await analyzeCommits(payload);
  }
};

module.exports = EventHandlers;
```

---

## ตัวอย่างจริง

### ตัวอย่างที่ 1: Auto-Fix Pull Request

```javascript
// scripts/auto-fix-pr.js

const { Octokit } = require("@octokit/rest");
const AISkills = require('../src');

async function autoFixPR(owner, repo, prNumber) {
  const octokit = new Octokit({
    auth: process.env.GITHUB_TOKEN
  });
  
  // ดึงไฟล์ที่เปลี่ยนแปลง
  const { data: files } = await octokit.pulls.listFiles({
    owner, repo, pull_number: prNumber
  });
  
  const fixer = new AISkills.PredictiveErrorFixerAgent({
    autoFix: true,
    iterateUntilClean: true
  });
  
  for (const file of files) {
    if (!file.filename.endsWith('.js')) continue;
    
    // ดึงเนื้อหาไฟล์
    const { data: content } = await octokit.repos.getContent({
      owner, repo, path: file.filename
    });
    
    const code = Buffer.from(content.content, 'base64').toString();
    
    // แก้ไข
    const result = await fixer.analyzeAndFixPreemptively(code);
    
    // บันทึกการเปลี่ยนแปลง
    console.log(`Fixed ${file.filename}: ${result.totalErrorsFixed} errors`);
  }
}

// ใช้งาน
autoFixPR('Nal2awit', 'my-repo', 42);
```

### ตัวอย่างที่ 2: Generate Tests Automatically

```javascript
// scripts/generate-tests-for-pr.js

const AISkills = require('../src');
const fs = require('fs');

async function generateTestsForPR(prFilePath) {
  const code = fs.readFileSync(prFilePath, 'utf8');
  
  const testGen = new AISkills.TestGeneratorAgent();
  const tests = await testGen.generateComprehensiveTests(code, {
    coverage: 80,
    includeEdgeCases: true
  });
  
  // บันทึกไฟล์ test
  const testFile = prFilePath.replace('.js', '.test.js');
  fs.writeFileSync(testFile, tests.join('\n\n'));
  
  console.log(`✅ Generated ${tests.length} tests → ${testFile}`);
}

// ใช้งาน
generateTestsForPR('./src/calculator.js');
```

### ตัวอย่างที่ 3: Continuous Code Quality Check

```javascript
// scripts/daily-quality-check.js

const AISkills = require('../src');
const fs = require('fs');
const path = require('path');

async function dailyQualityCheck(srcDir) {
  const results = [];
  
  // สแกนไฟล์ทั้งหมด
  const files = getAllJSFiles(srcDir);
  
  const analyzer = new AISkills.LineByLineAnalyzer();
  
  for (const file of files) {
    const code = fs.readFileSync(file, 'utf8');
    const analysis = await analyzer.analyzeEveryStatement(code);
    
    results.push({
      file,
      errors: analysis.errors.length,
      warnings: analysis.warnings.length,
      qualityScore: analysis.qualityScore
    });
  }
  
  // สร้างรายงาน
  const report = {
    date: new Date().toISOString(),
    totalFiles: files.length,
    totalErrors: results.reduce((sum, r) => sum + r.errors, 0),
    averageQuality: (results.reduce((sum, r) => sum + r.qualityScore, 0) / results.length).toFixed(2),
    details: results
  };
  
  // บันทึก
  fs.writeFileSync(
    `./reports/quality-${Date.now()}.json`,
    JSON.stringify(report, null, 2)
  );
  
  console.log(`Daily quality check complete:`, report);
  return report;
}

// ใช้งาน
dailyQualityCheck('./src');
```

---

## Automation Workflow

### ขั้นตอนการทำงานอัตโนมัติ

```
Developer push code
        ↓
   [Webhook trigger]
        ↓
   AI Analysis Start
        ↓
   ┌─────────────────────┐
   │ Step 1: Analyze     │
   │ - Check errors      │
   │ - Check security    │
   │ - Check performance │
   └─────────────────────┘
        ↓
   ┌─────────────────────┐
   │ Step 2: Fix         │
   │ - Auto-fix errors   │
   │ - Validate fixes    │
   │ - Run tests         │
   └─────────────────────┘
        ↓
   ┌─────────────────────┐
   │ Step 3: Report      │
   │ - Generate report   │
   │ - Post to GitHub    │
   │ - Save logs         │
   └─────────────────────┘
        ↓
   Done ✅
```

### ตั้ง Automation ใน Package.json

```json
{
  "scripts": {
    "analyze": "node scripts/analyze.js",
    "auto-fix": "node scripts/auto-fix.js",
    "test": "jest",
    "quality-check": "node scripts/daily-quality-check.js",
    "generate-docs": "node scripts/generate-docs.js",
    "generate-tests": "node scripts/generate-tests.js",
    "start-server": "node ai-server.js",
    "webhook": "node github-webhook.js",
    "ci": "npm run analyze && npm run auto-fix && npm run test"
  }
}
```

---

## การตั้งค่า CI/CD

### GitHub Actions Workflow Example

```yaml
# .github/workflows/ai-continuous.yml

name: AI Continuous Analysis & Fix

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * *'  # ทุกวันเที่ยงคืน

jobs:
  ai-analysis:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Node.js
      uses: actions/setup-node@v2
      with:
        node-version: '16'
    
    - name: Install
      run: npm install
    
    - name: Step 1: Analyze Code
      run: npm run analyze
      continue-on-error: true
    
    - name: Step 2: Auto-Fix Errors
      run: npm run auto-fix
      continue-on-error: true
    
    - name: Step 3: Generate Tests
      run: npm run generate-tests
      continue-on-error: true
    
    - name: Step 4: Run Tests
      run: npm test
      continue-on-error: true
    
    - name: Step 5: Quality Report
      run: npm run quality-check
      continue-on-error: true
    
    - name: Step 6: Generate Documentation
      run: npm run generate-docs
      continue-on-error: true
    
    - name: Upload Results
      uses: actions/upload-artifact@v2
      with:
        name: ai-analysis-results
        path: ./reports/
    
    - name: Notify via Comment
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          const report = JSON.parse(fs.readFileSync('./reports/summary.json', 'utf8'));
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `## AI Analysis Report\n\n${JSON.stringify(report, null, 2)}`
          });
```

---

## การใช้งาน API

### Client Library Example

```javascript
// client.js - การเรียก AI API จากโค้ด

class AISkillsClient {
  constructor(baseUrl = 'http://localhost:3000', apiKey = null) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
  }
  
  async analyze(code, language = 'javascript') {
    const response = await fetch(`${this.baseUrl}/api/analyze`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
      },
      body: JSON.stringify({ code, language })
    });
    
    return await response.json();
  }
  
  async fix(code, language = 'javascript') {
    const response = await fetch(`${this.baseUrl}/api/fix`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
      },
      body: JSON.stringify({ code, language })
    });
    
    return await response.json();
  }
  
  async generateTests(code, language = 'javascript') {
    const response = await fetch(`${this.baseUrl}/api/generate-tests`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
      },
      body: JSON.stringify({ code, language })
    });
    
    return await response.json();
  }
  
  async reviewCode(code, language = 'javascript') {
    const response = await fetch(`${this.baseUrl}/api/review`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
      },
      body: JSON.stringify({ code, language })
    });
    
    return await response.json();
  }
}

// ใช้งาน
const client = new AISkillsClient('http://localhost:3000');

// วิเคราะห์โค้ด
const analysis = await client.analyze('function test() { return user.name; }');
console.log(analysis.errors);

// แก้ไขโค้ด
const fixed = await client.fix('function test() { return user.name; }');
console.log(fixed.fixedCode);

// สร้าง test
const tests = await client.generateTests('function add(a, b) { return a + b; }');
console.log(tests.tests);
```

---

## ขั้นตอนการ Deploy

### 1. Prepare Environment

```bash
# ตั้งค่าตัวแปร environment
export GITHUB_TOKEN=your_token
export GITHUB_WEBHOOK_SECRET=your_secret
export API_PORT=3000
```

### 2. Install & Build

```bash
npm install
npm run build
```

### 3. Start Services

```bash
# เปิด AI Server
npm run start-server &

# เปิด Webhook receiver
npm run webhook &

# เปิด GitHub Actions workflow
# (จะรันอัตโนมัติเมื่อ push)
```

### 4. Verify Setup

```bash
# ทดสอบ API
curl -X POST http://localhost:3000/api/analyze \
  -H "Content-Type: application/json" \
  -d '{"code": "function test() {}", "language": "javascript"}'

# ตรวจสอบ logs
tail -f logs/ai-server.log
tail -f logs/webhook.log
```

---

## Troubleshooting

### ปัญหา: AI ไม่ส่งผลลัพธ์

```javascript
// ตรวจสอบ logs
const fs = require('fs');
const logs = fs.readFileSync('./logs/error.log', 'utf8');
console.log(logs);

// รีเซ็ต
npm run restart
```

### ปัญหา: Webhook ไม่ทำงาน

```bash
# ตรวจสอบ webhook URL ถูกต้อง
# ตรวจสอบ secret key
# ตรวจสอบ firewall

# ใช้ ngrok เพื่อ debug
ngrok http 3000
```

---

## สรุป

ตารางความสะดวกของการใช้งาน:

| วิธี | ความยากง่าย | ความเร็ว | ความยืดหยุ่น |
|---|---|---|---|
| Manual (npm run) | ⭐ ง่ายมาก | ⭐ ช้า | ⭐⭐⭐ สูง |
| GitHub Actions | ⭐⭐ ปกติ | ⭐⭐⭐ เร็ว | ⭐⭐ ปกติ |
| API Integration | ⭐⭐⭐ ยากกว่า | ⭐⭐⭐ เร็วมาก | ⭐⭐⭐⭐ สูงมาก |
| Webhook | ⭐⭐⭐⭐ ยากที่สุด | ⭐⭐⭐ เร็วมาก | ⭐⭐⭐⭐⭐ สูงสุด |

**เลือกวิธีตามความต้องการของคุณ!** 🚀

---

**สร้างด้วย ❤️ เพื่อให้ AI ทำงานแทนคุณได้อย่างแท้จริง**
