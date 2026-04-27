(function() {
    const table = document.getElementById('queryData');
    if (!table) return console.error("找不到表格");

    const toHalf = (val) => {
        if (!val) return "";
        return String(val).replace(/[\uff01-\uff5e]/g, (char) => 
            String.fromCharCode(char.charCodeAt(0) - 0xfee0)
        ).replace(/\u3000/g, " ").trim();
    };

    // --- 核心修正：金額抓取邏輯 ---
    const cleanAmount = (cell) => {
        if (!cell) return "";
        
        // 1. 抓取所有子節點的內容並過濾空白
        let parts = Array.from(cell.childNodes)
                         .map(node => node.textContent.trim())
                         .filter(text => text !== "");

        // 2. 根據規律：第一個 part 通常是「處理序號」
        // 如果有複數個節點，我們移除第一個，並合併剩下的
        if (parts.length > 1) {
            parts.shift(); // 移除序號
            let amountStr = parts.join(""); // 合併剩下的部分 (解決 371 + 453 的問題)
            return toHalf(amountStr).replace(/[^0-9.]/g, ""); // 只留數字與小數點
        }

        // 3. 如果只有一個節點，且長度大於 4 (包含序號黏在一起的情況)
        let singleText = toHalf(parts[0] || "");
        // 這種情況通常序號是 1 位數，後面是金額。如果確定金額都有千分位，
        // 且你的「正確數字」第一位跟「抓出結果」第一位相同，代表序號佔了一位。
        if (singleText.length > 1) {
            // 移除第一個字元 (序號) 並清理
            return singleText.substring(1).replace(/[^0-9.]/g, "");
        }

        return singleText.replace(/[^0-9.]/g, "");
    };

    const rows = Array.from(table.rows);
    const resultData = [];
    const cleanHeaders = ["生效日", "合約編號", "轉出帳號", "轉入帳號", "總金額", "處理狀態", "備註附言"];
    resultData.push(cleanHeaders.join(","));

    let startProcessing = false;

    for (let i = 0; i < rows.length; i++) {
        const row = rows[i];
        if (!row.cells || row.cells.length < 10) continue;

        const cells = Array.from(row.cells);
        const cellTexts = cells.map(c => c.innerText.trim());

        // 自動偵測起始列
        if (!startProcessing) {
            if (cellTexts[2] && (cellTexts[2].includes('/') || cellTexts[2].includes('202'))) {
                startProcessing = true;
            } else continue;
        }

        const mappedRow = [
            toHalf(cellTexts[2]).replace(/\s+/g, ""),             // 生效日
            "\t" + toHalf(cellTexts[3]).replace(/\s+/g, ""),      // 合約編號
            "\t" + toHalf(cellTexts[4]).replace(/\s+/g, ""),      // 轉出帳號
            // 轉入帳號處理：去尾
            ((val) => {
                let s = toHalf(val).replace(/\s+/g, "");
                if (s.includes('-')) {
                    let p = s.split('-');
                    p.pop();
                    s = p.join('-');
                }
                return "\t" + s;
            })(cellTexts[5]),
            cleanAmount(cells[6]),                            // 總金額 (核心修正)
            toHalf(cellTexts[8]),                                 // 處理狀態
            toHalf(cellTexts[10]).replace(/[\r\n\s]+/g, "").replace(/,/g, "，") // 備註
        ];

        resultData.push(mappedRow.join(","));
    }

    const csvString = "\ufeff" + resultData.join("\n");
    const blob = new Blob([csvString], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const link = document.createElement("a");
    link.href = url;
    link.download = `CTBT_Banklist.csv`;
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);

    console.log("金額精準修復完成。");
})();