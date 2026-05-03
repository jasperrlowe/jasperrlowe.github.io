---
title: Projects
date: 2025-10-02 00:00:00
type: "projects"
comments: false
---

<style>
.project-card {
  background: #fff;
  border: 2px solid #e5e7eb;
  border-radius: 12px;
  padding: 24px;
  margin-bottom: 32px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
  transition: all 0.3s ease;
}

.project-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  border-color: #3b82f6;
}

.project-header {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 16px;
}

.project-icon {
  width: 60px;
  height: 60px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 28px;
  flex-shrink: 0;
}

.project-title {
  font-size: 24px;
  font-weight: 700;
  color: #1f2937;
  margin: 0;
}

.project-subtitle {
  font-size: 14px;
  color: #6b7280;
  margin: 4px 0 0 0;
}

.project-description {
  color: #4b5563;
  line-height: 1.7;
  margin-bottom: 20px;
  font-size: 15px;
}

.project-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 12px;
  margin-bottom: 20px;
}

.meta-item {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 14px;
  color: #6b7280;
}

.tech-stack {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 20px;
}

.tech-badge {
  display: inline-block;
  padding: 4px 12px;
  background: #eff6ff;
  color: #1e40af;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 600;
  border: 1px solid #bfdbfe;
}

.project-links {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

.project-btn {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 10px 20px;
  border-radius: 8px;
  font-weight: 600;
  font-size: 14px;
  text-decoration: none;
  transition: all 0.2s;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: 2px solid #3b82f6;
}

.btn-primary:hover {
  background: #2563eb;
  border-color: #2563eb;
  transform: translateY(-2px);
  color: white;
}

.btn-secondary {
  background: white;
  color: #3b82f6;
  border: 2px solid #3b82f6;
}

.btn-secondary:hover {
  background: #eff6ff;
  transform: translateY(-2px);
  color: #2563eb;
}

.project-status {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 12px;
  background: #dcfce7;
  color: #166534;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 600;
  border: 1px solid #86efac;
  margin-bottom: 16px;
}

.status-dot {
  width: 8px;
  height: 8px;
  background: #22c55e;
  border-radius: 50%;
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.highlight-box {
  background: #fef3c7;
  border-left: 4px solid #f59e0b;
  padding: 16px;
  margin: 20px 0;
  border-radius: 6px;
}

.highlight-box strong {
  color: #92400e;
}

@media (max-width: 768px) {
  .project-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .project-icon {
    width: 50px;
    height: 50px;
    font-size: 24px;
  }

  .project-title {
    font-size: 20px;
  }
}
</style>

## 🚀 My Engineering Projects

Welcome to my project showcase! Here you'll find the technical systems I've built, each with detailed case studies and open-source code.

---

<div class="project-card">
  <div class="project-status">
    <span class="status-dot"></span>
    Active Development
  </div>

  <div class="project-header">
    <div class="project-icon">🎓</div>
    <div>
      <h3 class="project-title">Holistic Assistant</h3>
      <p class="project-subtitle">RAG-Powered Academic Advisor for CUHK-SZ Students</p>
    </div>
  </div>

  <p class="project-description">
    A comprehensive AI-driven academic planning system that integrates fragmented course information from 36 training scheme PDFs and the SIS portal. Provides 24/7 personalized course consultation based on official data, helping 1,500+ students make informed academic decisions.
  </p>

  <div style="margin: 20px 0;">
    <div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr)); gap: 16px;">
      <div style="background: #f3f4f6; border: 2px dashed #d1d5db; border-radius: 8px; padding: 60px 20px; text-align: center; color: #9ca3af;">
        <div style="font-size: 48px; margin-bottom: 12px;">🎨</div>
        <div style="font-weight: 600; color: #6b7280;">Orbit UI Interface</div>
        <div style="font-size: 13px; margin-top: 4px;">Planet-style navigation system</div>
      </div>
      <div style="background: #f3f4f6; border: 2px dashed #d1d5db; border-radius: 8px; padding: 60px 20px; text-align: center; color: #9ca3af;">
        <div style="font-size: 48px; margin-bottom: 12px;">💬</div>
        <div style="font-weight: 600; color: #6b7280;">AI Chat Interface</div>
        <div style="font-size: 13px; margin-top: 4px;">Context-aware course consultation</div>
      </div>
      <div style="background: #f3f4f6; border: 2px dashed #d1d5db; border-radius: 8px; padding: 60px 20px; text-align: center; color: #9ca3af;">
        <div style="font-size: 48px; margin-bottom: 12px;">📊</div>
        <div style="font-weight: 600; color: #6b7280;">Planning Generator</div>
        <div style="font-size: 13px; margin-top: 4px;">Semester course recommendations</div>
      </div>
    </div>
    <div style="text-align: center; margin-top: 12px; font-size: 13px; color: #9ca3af;">
      📸 Screenshots coming soon - Project currently in private beta
    </div>
  </div>

  <div class="highlight-box">
    <strong>Key Achievement:</strong> Reduced course planning time from 45 minutes to <2 seconds per query, with 95.2% data coverage and 96.8% extraction accuracy.
  </div>

  <div class="project-meta">
    <div class="meta-item">
      <span>📅</span>
      <span>Oct 2025 - Dec 2025</span>
    </div>
    <div class="meta-item">
      <span>🎯</span>
      <span>1,568 Courses × 3 Semesters</span>
    </div>
    <div class="meta-item">
      <span>⚡</span>
      <span>P50 Response: 1.7s</span>
    </div>
  </div>

  <div class="tech-stack">
    <span class="tech-badge">Flask</span>
    <span class="tech-badge">Gemini 2.5 Flash</span>
    <span class="tech-badge">Playwright</span>
    <span class="tech-badge">RAG</span>
    <span class="tech-badge">PyPDF2</span>
    <span class="tech-badge">Vanilla JS</span>
    <span class="tech-badge">Tailwind CSS</span>
  </div>

  <div class="project-links">
    <a href="/2025/12/24/holistic-assistant-case-study/" class="project-btn btn-primary">
      📖 Read Case Study
    </a>
    <a href="/2025/11/20/holistic-assistant-mvp/" class="project-btn btn-secondary">
      🔬 MVP Report
    </a>
    <a href="https://github.com/Jerry0310no1/holistic-assistant" class="project-btn btn-secondary" target="_blank">
      💻 View Code
    </a>
  </div>

  <details style="margin-top: 20px;">
    <summary style="cursor: pointer; color: #3b82f6; font-weight: 600;">📊 Technical Highlights</summary>
    <ul style="margin-top: 12px; color: #4b5563; line-height: 1.8;">
      <li><strong>Hybrid Context RAG:</strong> File-based retrieval + hardcoded templates (no Vector DB needed for <50MB data)</li>
      <li><strong>Smart Model Routing:</strong> Keyword-based routing between Gemini Flash (1.5s) and Pro (4s), achieving 42% cost reduction</li>
      <li><strong>Anti-Bot Scraping:</strong> Playwright with session persistence, throttling, and retry mechanisms (99.2% success rate)</li>
      <li><strong>AI-Powered PDF Extraction:</strong> Gemini Vision API reduced error rate from 40% to 5% vs. regex-only approach</li>
      <li><strong>Windows Multiprocessing:</strong> Hard timeout mechanism using Process + Pipe to handle Gemini API hangs</li>
    </ul>
  </details>
</div>

---

<div style="text-align: center; padding: 40px 0; color: #6b7280;">
  <p style="font-size: 18px; margin-bottom: 12px;">✨ More projects coming soon...</p>
  <p style="font-size: 14px;">I'm currently exploring distributed systems, LLM agents, and real-time data pipelines.</p>
</div>

---

## 📬 Get in Touch

Interested in collaborating or discussing technical ideas?

**Email:** [125090445@link.cuhk.edu.cn](mailto:125090445@link.cuhk.edu.cn)
**GitHub:** [@Jerry0310no1](https://github.com/Jerry0310no1)
**Blog:** [jerry0310no1.github.io](https://jerry0310no1.github.io)
