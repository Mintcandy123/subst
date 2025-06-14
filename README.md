<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8" />
  <title>암호화</title>
  <style>
    body {
      font-family: 'Arial', sans-serif;
      line-height: 1.6;
      margin: 20px;
      background-color: #f4f4f4;
      color: #333;
    }
    h2 {
      color: #5a20c7;
    }
    input[type="text"], textarea, select {
      padding: 10px;
      width: 100%;
      box-sizing: border-box;
      font-size: 16px;
      margin-bottom: 5px;
    }
    button {
      padding: 10px 20px;
      margin: 5px 10px 10px 0;
      background-color: #5a20c7;
      color: white;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-size: 16px;
    }
    button:hover {
      background-color: #45179e;
    }
    #clearButton {
      background-color: #5a20c7;
      color: white;
    }
    #clearButton:hover {
      background-color: #45179e;
    }
    #displayArea {
      margin-top: 20px;
      background: #e9e2f7;
      border: 1px dashed #5a20c7;
      padding: 10px;
      min-height: 50px;
      white-space: pre-wrap;
      border-radius: 4px;
      margin-bottom: 10px;
    }
    #copyButton {
      background-color: #28a745;
    }
    #copyButton:hover {
      background-color: #218838;
    }
    #ruleEditorModal {
      display: none;
      position: fixed;
      top: 20%;
      left: 50%;
      transform: translateX(-50%);
      background: #fff;
      border: 1px solid #ccc;
      padding: 20px;
      z-index: 1000;
      box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
      border-radius: 8px;
      width: 400px;
    }
    #ruleEditorModal h3 {
      color: #5a20c7;
      margin-top: 0;
    }
  </style>
</head>
<body>

  <h2>암호화 🔐</h2>

  <label for="textToProcess">입력 텍스트:</label><br />
  <input type="text" id="textToProcess" placeholder="예: 안녕" />
  <button id="clearButton" onclick="clearInput()">❌ 지우기</button>

  <br /><br />

  <label for="ruleSetSelect">규칙 세트 선택:</label>
  <select id="ruleSetSelect"></select>
  <button onclick="addRuleSet()">세트 추가</button>
  <button onclick="editRuleSet()">세트 수정</button><br />
  <small>※ 동일한 치환 규칙이 있으면, <strong>뒤에 있는 규칙이 우선 적용</strong>됩니다.</small><br />
  <small>※ ||| 또는 %%% 를 변환 규칙에 지정, 암호화 밑 복호화 할 시 오류가 발생할 수 있습니다.</small>

  <br /><br />
  <button onclick="encryptText()">암호화</button>
  <button onclick="decryptText()">복호화</button>

  <div id="displayArea">결과가 여기에 표시됩니다.</div>
  <button id="copyButton" onclick="copyResult()">결과 복사</button>

  <div id="ruleEditorModal">
    <h3>규칙 세트 편집</h3>
    <label>세트 이름:<br />
      <input type="text" id="ruleSetName" />
    </label><br /><br />
    <label>치환 규칙 (예: ㄱ=1,ㅏ=0,ㄴ=2):<br />
      <textarea id="ruleSetContent" rows="4"></textarea>
    </label><br /><br />
    <button onclick="saveRuleSet()">저장</button>
    <button onclick="deleteRuleSet()">삭제</button>
    <button onclick="closeRuleEditor()">닫기</button>
  </div>

  <script>
    const CHO = [...'ㄱㄲㄴㄷㄸㄹㅁㅂㅃㅅㅆㅇㅈㅉㅊㅋㅌㅍㅎ'];
    const JUNG = [...'ㅏㅐㅑㅒㅓㅔㅕㅖㅗㅘㅙㅚㅛㅜㅝㅞㅟㅠㅡㅢㅣ'];
    const JONG = ['',...'ㄱㄲㄳㄴㄵㄶㄷㄹㄺㄻㄼㄽㄾㄿㅀㅁㅂㅄㅅㅆㅇㅈㅊㅋㅌㅍㅎ'];

    const disassembleHangul = str => [...str].map(char => {
      const code = char.charCodeAt(0);
      if (code < 0xAC00 || code > 0xD7A3) return [char];
      const i = code - 0xAC00, cho = Math.floor(i / 588), jung = Math.floor((i % 588) / 28), jong = i % 28;
      return [CHO[cho], JUNG[jung], JONG[jong]].filter(c => c);
    });

    const assembleHangul = arr => arr.map(parts => {
      if (parts.length === 1 && !CHO.includes(parts[0]) && !JUNG.includes(parts[0])) return parts[0];
      const c = CHO.indexOf(parts[0]), j = JUNG.indexOf(parts[1]), g = parts[2] ? JONG.indexOf(parts[2]) : 0;
      return (c < 0 || j < 0) ? parts.join('') : String.fromCharCode(0xAC00 + c * 588 + j * 28 + g);
    }).join('');

    const parseRules = str => new Map(
      str.split(';').map(r => r.split('=').map(s => s.trim())).filter(([k, v]) => k && v)
    );
    const createReverseRules = m => new Map([...m.entries()].map(([k, v]) => [v, k]));

    function getAllRuleSets() {
      return JSON.parse(localStorage.getItem("ruleSets") || "{}");
    }

    function saveAllRuleSets(data) {
      localStorage.setItem("ruleSets", JSON.stringify(data));
    }

    function loadRuleSets() {
      const sets = getAllRuleSets();
      const sel = document.getElementById('ruleSetSelect');
      sel.innerHTML = '';
      Object.keys(sets).forEach(name => {
        const opt = document.createElement('option');
        opt.value = name;
        opt.textContent = name;
        sel.appendChild(opt);
      });
      if (sel.options.length) sel.selectedIndex = 0;
    }

    function getSelectedRules() {
      const name = document.getElementById('ruleSetSelect').value;
      const sets = getAllRuleSets();
      return parseRules(sets[name] || '');
    }

    function addRuleSet() {
      document.getElementById('ruleSetName').value = '';
      document.getElementById('ruleSetContent').value = '';
      document.getElementById('ruleEditorModal').style.display = 'block';
    }

    function editRuleSet() {
      const name = document.getElementById('ruleSetSelect').value;
      const sets = getAllRuleSets();
      document.getElementById('ruleSetName').value = name;
      document.getElementById('ruleSetContent').value = sets[name] || '';
      document.getElementById('ruleEditorModal').style.display = 'block';
    }

    function closeRuleEditor() {
      document.getElementById('ruleEditorModal').style.display = 'none';
    }

    function saveRuleSet() {
      const name = document.getElementById('ruleSetName').value.trim();
      const content = document.getElementById('ruleSetContent').value.trim();
      if (!name) return alert('세트 이름을 입력하세요.');

      const sets = getAllRuleSets();
      if (!(name in sets) && Object.keys(sets).length >= 15)
        return alert('세트는 최대 15개까지 저장할 수 있습니다.');

      sets[name] = content;
      saveAllRuleSets(sets);
      loadRuleSets();
      document.getElementById('ruleSetSelect').value = name;
      closeRuleEditor();
    }

    function deleteRuleSet() {
      const name = document.getElementById('ruleSetName').value.trim();
      if (!confirm(`"${name}" 세트를 삭제할까요?`)) return;
      const sets = getAllRuleSets();
      delete sets[name];
      saveAllRuleSets(sets);
      loadRuleSets();
      closeRuleEditor();
    }

    function encryptText() {
      const text = document.getElementById('textToProcess').value;
      const rules = getSelectedRules();
      const disassembled = disassembleHangul(text);
      const encrypted = disassembled.map(g =>
        g.map(j => rules.has(j) ? `[[${rules.get(j)}]]` : j).join(',')
      ).join("'");
      document.getElementById('displayArea').textContent = encrypted;
    }

    function decryptText() {
      const encrypted = document.getElementById('textToProcess').value;
      const rules = getSelectedRules();
      const rev = createReverseRules(rules);
      const result = assembleHangul(
        encrypted.split("'").map(part =>
          part.split(',').map(tok => {
            const match = tok.match(/^\[\[(.*?)\]\]$/);
            return match ? (rev.get(match[1]) || match[1]) : tok;
          })
        )
      );
      document.getElementById('displayArea').textContent = result;
    }

    function clearInput() {
      document.getElementById('textToProcess').value = '';
    }

    async function copyResult() {
      const text = document.getElementById('displayArea').textContent;
      if (!text || text === "결과가 여기에 표시됩니다.") return;
      try {
        await navigator.clipboard.writeText(text);
        const btn = document.getElementById('copyButton');
        const original = btn.textContent;
        btn.textContent = '✅ 복사됨!';
        btn.disabled = true;
        setTimeout(() => {
          btn.textContent = original;
          btn.disabled = false;
        }, 1500);
      } catch {
        alert("복사 실패 😥");
      }
    }

    window.addEventListener('load', loadRuleSets);
  </script>

</body>
</html>
