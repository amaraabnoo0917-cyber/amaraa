<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>교안 스마트 요약 및 예상문제 생성기</title>
    <!-- Tailwind CSS CDN -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- PDF.js (브라우저 PDF 텍스트 추출용) -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
    <script>
        pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
    </script>
</head>
<body class="bg-slate-50 min-h-screen text-slate-800">

    <div class="max-w-4xl mx-auto px-4 py-8">
        <!-- 헤더 -->
        <header class="text-center mb-10">
            <h1 class="text-3xl font-extrabold text-indigo-600 mb-2">📚 교안 AI 요약 & 예상문제 생성기</h1>
            <p class="text-slate-600">교안 PDF를 업로드하면 핵심 요약과 4지선다 10문제를 자동으로 만들어 드립니다.</p>
        </header>

        <!-- API 키 설정 카드 -->
        <section class="bg-white rounded-2xl shadow-sm p-6 mb-6 border border-slate-100">
            <div class="flex flex-col md:flex-row items-center justify-between gap-4">
                <div>
                    <h2 class="font-bold text-lg text-slate-700">🔑 Gemini API 키 설정</h2>
                    <p class="text-sm text-slate-500">Google AI Studio에서 발급받은 API 키(AIzaSy...)를 입력해 주세요.</p>
                </div>
                <div class="flex w-full md:w-auto gap-2">
                    <input type="password" id="apiKeyInput" placeholder="AIzaSy..." class="px-4 py-2 border rounded-xl w-full md:w-80 focus:outline-none focus:ring-2 focus:ring-indigo-500 text-sm">
                    <button onclick="saveApiKey()" class="bg-indigo-600 hover:bg-indigo-700 text-white font-semibold px-4 py-2 rounded-xl text-sm transition whitespace-nowrap">저장</button>
                </div>
            </div>
        </section>

        <!-- 파일 업로드 카드 -->
        <section class="bg-white rounded-2xl shadow-sm p-6 mb-8 border border-slate-100">
            <h2 class="font-bold text-lg text-slate-700 mb-4">📂 교안 PDF 첨부</h2>
            <div class="border-2 border-dashed border-slate-200 rounded-xl p-8 text-center hover:border-indigo-400 transition cursor-pointer" onclick="document.getElementById('pdfFile').click()">
                <input type="file" id="pdfFile" accept="application/pdf" class="hidden" onchange="handleFileSelect(event)">
                <div id="fileInfo" class="text-slate-500">
                    <svg class="mx-auto h-12 w-12 text-slate-400 mb-3" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12"></path>
                    </svg>
                    <span class="font-medium text-indigo-600">파일을 선택하려면 여기를 누르세요</span> 또는 드래그하여 업로드하세요.
                </div>
            </div>

            <button id="generateBtn" onclick="processPdfAndGenerate()" disabled class="mt-6 w-full bg-slate-300 text-slate-500 font-bold py-3 rounded-xl transition cursor-not-allowed">
                AI 분석 및 문제 생성 시작
            </button>
        </section>

        <!-- 로딩 표시 -->
        <div id="loading" class="hidden text-center py-12">
            <div class="inline-block animate-spin rounded-full h-12 w-12 border-4 border-indigo-500 border-t-transparent mb-4"></div>
            <p id="loadingText" class="text-lg font-medium text-slate-600">PDF 텍스트를 추출하고 있습니다...</p>
        </div>

        <!-- 결과 영역 -->
        <div id="resultSection" class="hidden space-y-8">
            <!-- 요약 결과 -->
            <section class="bg-white rounded-2xl shadow-sm p-6 border border-slate-100">
                <h3 class="text-xl font-bold text-indigo-600 mb-4 flex items-center gap-2">
                    <span>📌 교안 핵심 요약</span>
                </h3>
                <div id="summaryContent" class="prose max-w-none text-slate-700 leading-relaxed whitespace-pre-line bg-slate-50 p-5 rounded-xl"></div>
            </section>

            <!-- 예상문제 결과 -->
            <section class="bg-white rounded-2xl shadow-sm p-6 border border-slate-100">
                <h3 class="text-xl font-bold text-indigo-600 mb-4 flex items-center gap-2">
                    <span>📝 AI 추천 4지선다 예상문제 (10문항)</span>
                </h3>
                <div id="quizContent" class="space-y-6"></div>
            </section>
        </div>
    </div>

    <!-- Official Google Gen AI Web SDK Module Import -->
    <script type="module">
        import { GoogleGenAI } from "https://esm.run/@google/genai";

        window.GoogleGenAI = GoogleGenAI;
    </script>

    <script>
        let extractedText = "";

        window.onload = function() {
            const savedKey = localStorage.getItem('gemini_api_key');
            if (savedKey) {
                document.getElementById('apiKeyInput').value = savedKey;
            }
        };

        function saveApiKey() {
            const key = document.getElementById('apiKeyInput').value.trim();
            if (!key) {
                alert('API 키를 입력해주세요.');
                return;
            }
            localStorage.setItem('gemini_api_key', key);
            alert('API 키가 성공적으로 저장되었습니다!');
        }

        async function handleFileSelect(event) {
            const file = event.target.files[0];
            if (!file) return;

            if (file.type !== 'application/pdf') {
                alert('PDF 파일만 업로드 가능합니다.');
                return;
            }

            document.getElementById('fileInfo').innerHTML = `<p class="font-semibold text-indigo-600">선택된 파일: ${file.name}</p>`;
            showLoading(true, "PDF 파일을 읽는 중입니다...");
            
            try {
                const arrayBuffer = await file.arrayBuffer();
                const pdf = await pdfjsLib.getDocument({ data: arrayBuffer }).promise;
                let fullText = "";

                for (let i = 1; i <= pdf.numPages; i++) {
                    const page = await pdf.getPage(i);
                    const textContent = await page.getTextContent();
                    const pageText = textContent.items.map(item => item.str).join(' ');
                    fullText += `[페이지 ${i}]\n` + pageText + "\n\n";
                }

                extractedText = fullText;
                showLoading(false);
                
                if (extractedText.trim().length > 0) {
                    const btn = document.getElementById('generateBtn');
                    btn.disabled = false;
                    btn.classList.remove('bg-slate-300', 'text-slate-500', 'cursor-not-allowed');
                    btn.classList.add('bg-indigo-600', 'hover:bg-indigo-700', 'text-white', 'cursor-pointer');
                } else {
                    alert('PDF에서 텍스트를 추출할 수 없습니다. 이미지 스캔 파일인지 확인해 주세요.');
                }
            } catch (error) {
                showLoading(false);
                alert('PDF를 읽는 중 오류가 발생했습니다: ' + error.message);
            }
        }

        function showLoading(show, text = "") {
            const loading = document.getElementById('loading');
            if (show) {
                document.getElementById('loadingText').innerText = text;
                loading.classList.remove('hidden');
                document.getElementById('resultSection').classList.add('hidden');
            } else {
                loading.classList.add('hidden');
            }
        }

        async function processPdfAndGenerate() {
            const apiKey = localStorage.getItem('gemini_api_key') || document.getElementById('apiKeyInput').value.trim();
            if (!apiKey) {
                alert('먼저 Gemini API 키를 입력하고 저장해주세요.');
                return;
            }

            if (!extractedText) {
                alert('업로드된 교안 텍스트가 없습니다.');
                return;
            }

            showLoading(true, "Gemini AI가 교안을 분석하고 예상문제를 출제 중입니다...");

            try {
                const ai = new window.GoogleGenAI({ apiKey: apiKey });

                const prompt = `
당신은 전문 교육 강사 및 출제자입니다. 아래 제공된 교안 내용을 바탕으로 두 가지 작업을 수행해 주세요.

1. 교안의 핵심 내용을 체계적이고 알기 쉽게 요약해 주세요.
2. 교안 내용을 바탕으로 핵심을 평가할 수 있는 객관식 4지선다 예상문제 **정확히 10문제**를 출제해 주세요.

반드시 아래의 **JSON 형식**으로만 응답해 주세요. 마크다운 백틱이나 다른 설명글을 절대 포함하지 말고 pure JSON 문자열만 반환해야 합니다.

{
  "summary": "교안 전체 요약 내용",
  "questions": [
    {
      "id": 1,
      "question": "문제 내용",
      "options": ["1번 보기", "2번 보기", "3번 보기", "4번 보기"],
      "answer": 0,
      "explanation": "정답 해설"
    }
  ]
}

교안 내용:
${extractedText.substring(0, 30000)}
                `;

                // Latest Supported Model Update
                const response = await ai.models.generateContent({
                    model: 'gemini-3.8-flash',
                    contents: prompt,
                });

                const rawText = response.text;
                
                const jsonMatch = rawText.match(/\{[\s\S]*\}/);
                if (!jsonMatch) {
                    throw new Error("AI 응답을 JSON으로 파싱할 수 없습니다.");
                }

                const resultData = JSON.parse(jsonMatch[0]);

                displayResults(resultData);
                showLoading(false);
                document.getElementById('resultSection').classList.remove('hidden');

            } catch (error) {
                showLoading(false);
                alert('AI 생성 중 오류가 발생했습니다: ' + error.message);
            }
        }

        function displayResults(data) {
            document.getElementById('summaryContent').innerText = data.summary;

            const quizContainer = document.getElementById('quizContent');
            quizContainer.innerHTML = '';

            data.questions.forEach((q, idx) => {
                let optionsHtml = '';
                q.options.forEach((opt, optIdx) => {
                    optionsHtml += `
                        <label class="flex items-center gap-3 p-3 rounded-xl border border-slate-200 hover:bg-slate-50 cursor-pointer transition">
                            <input type="radio" name="question_${idx}" value="${optIdx}" class="text-indigo-600 focus:ring-indigo-500">
                            <span class="text-slate-700 text-sm">${opt}</span>
                        </label>
                    `;
                });

                const qDiv = document.createElement('div');
                qDiv.className = 'bg-slate-50 p-6 rounded-xl border border-slate-200';
                qDiv.innerHTML = `
                    <h4 class="font-bold text-slate-800 mb-3">Q${idx + 1}. ${q.question}</h4>
                    <div class="space-y-2 mb-4">${optionsHtml}</div>
                    <button onclick="toggleAnswer(${idx})" class="text-xs font-semibold text-indigo-600 hover:text-indigo-800 underline">정답 및 해설 보기</button>
                    <div id="explanation_${idx}" class="hidden mt-3 p-4 bg-indigo-50 rounded-xl text-sm text-indigo-900 border border-indigo-100">
                        <p class="font-bold mb-1">정답: ${q.options[q.answer]}</p>
                        <p>${q.explanation}</p>
                    </div>
                `;
                quizContainer.appendChild(qDiv);
            });
        }

        function toggleAnswer(idx) {
            const expDiv = document.getElementById(`explanation_${idx}`);
            expDiv.classList.toggle('hidden');
        }
    </script>
</body>
</html>
