# Youning1230.github.io
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>导入图片一键生成闭合轨迹工具（适合简单图形）v14</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
            font-family: "Microsoft Yahei", sans-serif;
        }
        body {
            background: #ffffff;
            color: #222222;
            padding: 16px;
        }
        h1 {
            text-align: center;
            margin-bottom: 18px;
            font-size: 22px;
            color: #111;
        }
        /* 三栏横向主布局 */
        .main-wrap {
            display: grid;
            grid-template-columns: 320px 380px 1fr;
            gap: 16px;
            max-width: 1800px;
            margin: 0 auto;
        }
        .card {
            background: #f6f8fa;
            padding: 18px;
            border-radius: 10px;
            border: 1px solid #e1e4e8;
            height: fit-content;
        }
        h2 {
            margin-bottom: 12px;
            color: #2563eb;
            border-bottom: 1px solid #d0d7e0;
            padding-bottom: 5px;
            font-size: 16px;
        }
        h3 {
            font-size: 14px;
            color: #333;
            margin:12px 0 6px;
        }
        .row {
            margin: 9px 0;
            display: flex;
            gap: 8px;
            align-items: center;
        }
        label {
            width: 140px;
            font-size: 14px;
        }
        input[type="number"],
        input[type="file"] {
            padding: 6px 9px;
            background: #ffffff;
            border: 1px solid #cbd5e1;
            color: #111;
            border-radius: 6px;
            width: 160px;
            font-size: 14px;
        }
        .btn-group {
            display: flex;
            gap:8px;
            margin-bottom:10px;
            flex-wrap: wrap;
        }
        button {
            padding: 8px 14px;
            background: #2563eb;
            color: #fff;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 13px;
            transition: 0.2s;
        }
        .btn-note-free {
            background:#0284c7;
        }
        .btn-download-mod {
            background:#059669;
        }
        button:hover {
            background: #1d4ed8;
        }
        .btn-note-free:hover {
            background:#0369a1;
        }
        .btn-download-mod:hover {
            background:#047857;
        }
        canvas {
            border: 1px solid #cbd5e1;
            background: #fff;
            width: 100%;
            max-height: 340px;
            margin-top:4px;
            display:block;
        }
        #codeOutput {
            width: 100%;
            min-height: 580px;
            background: #0f172a;
            color: #4ade80;
            padding: 10px;
            border: 1px solid #cbd5e1;
            border-radius: 8px;
            white-space: pre;
            overflow: auto;
            font-size: 12px;
        }
        .tip {
            color: #64748b;
            font-size: 12px;
            margin-top: 3px;
        }
        .seg-desc {
            display: none;
        }
    </style>
</head>
<body>
    <h1>张苑的导入图片一键生成闭合轨迹工具（适合简单图形）v14</h1>

    <!-- 三栏横向布局：左参数原图 | 中轮廓预览 | 右代码区 -->
    <div class="main-wrap">
        <!-- 左栏：参数设置 + 完整适配原图预览 -->
        <div class="card">
            <h2>图像上传 & 绘图参数(mm)</h2>
            <div class="row">
                <label>上传线条图：</label>
                <input type="file" id="imgFile" accept="image/png,image/jpg,image/jpeg">
            </div>
            <div class="row">
                <label>绘图宽度 W(mm)：</label>
                <input type="number" id="drawW" value="200" min="10" max="1000">
            </div>
            <div class="row">
                <label>绘图高度 H(mm)：</label>
                <input type="number" id="drawH" value="200" min="10" max="1000">
            </div>
            <div class="row">
                <label>轮廓抽稀精度(像素)：</label>
                <input type="number" id="simplifyTol" value="2.5" min="0.5" max="10" step="0.1">
            </div>
            <div class="row">
                <label>最小轮廓过滤：</label>
                <input type="number" id="minContourLen" value="60" min="10" max="300">
            </div>
            <div class="tip">抬落笔20mm | Tool0 wobj1 | 图片完整显示无裁切</div>
            <div class="row" style="margin-top:14px">
                <button id="genTraj">生成ABB轨迹代码</button>
            </div>
            <h3>原图预览（完整适配框内）</h3>
            <canvas id="canvasIn"></canvas>
            <div class="seg-desc" id="segInfo"></div>
        </div>

        <!-- 中间栏：平滑轮廓预览（纯白背景，仅显示轮廓，不显示原图） -->
        <div class="card">
            <h2>平滑边缘轮廓预览（仅轨迹线条）</h2>
            <canvas id="canvasOut"></canvas>
        </div>

        <!-- 右侧整栏：代码输出与操作按钮 -->
        <div class="card">
            <h2>ABB机器人RAPID绘图程序（Movel）</h2>
            <div class="btn-group">
                <button id="copyCode">复制带注释程序</button>
                <button id="copyCodeNoNote" class="btn-note-free">复制无注释纯代码</button>
                <button id="downloadMod" class="btn-download-mod">下载MOD文件</button>
            </div>
            <textarea id="codeOutput" readonly></textarea>
        </div>
    </div>

    <script>
        let originImg = null;
        let contourList = [];
        let fullRapidText = "";
        const penUpOffset = 20;
        const safeZTop = 100;
        const colorList = ['#ef4444','#22c55e','#3b82f6','#eab308','#a855f7','#f97316','#06b6d4'];

        const cvsIn = document.getElementById("canvasIn");
        const ctxIn = cvsIn.getContext("2d");
        const cvsOut = document.getElementById("canvasOut");
        const ctxOut = cvsOut.getContext("2d");
        const codeBox = document.getElementById("codeOutput");
        const segInfoDom = document.getElementById("segInfo");
        const inputDrawW = document.getElementById("drawW");
        const inputDrawH = document.getElementById("drawH");

        // 原图渲染：完整居中显示在画布内，无裁切
        function renderFitCanvas(targetCanvas, targetCtx, sourceImg, containerWidth, maxHeight) {
            if (!sourceImg) return null;
            const drawW = Number(inputDrawW.value);
            const drawH = Number(inputDrawH.value);
            const imgRawW = sourceImg.width;
            const imgRawH = sourceImg.height;

            const mmRatio = drawW / drawH;
            const cW = containerWidth;
            const cH = maxHeight;
            const cRatio = cW / cH;

            let renderW, renderH;
            if (mmRatio > cRatio) {
                renderW = cW;
                renderH = cW / mmRatio;
            } else {
                renderH = cH;
                renderW = cH * mmRatio;
            }

            targetCanvas.width = Math.round(cW);
            targetCanvas.height = Math.round(cH);
            targetCtx.fillStyle = "#ffffff";
            targetCtx.fillRect(0, 0, cW, cH);
            const offsetX = (cW - renderW) / 2;
            const offsetY = (cH - renderH) / 2;
            targetCtx.drawImage(sourceImg, offsetX, offsetY, renderW, renderH);

            return {
                mmScaleX: imgRawW / drawW,
                mmScaleY: imgRawH / drawH,
                renderW, renderH,
                offsetX, offsetY,
                previewPixelPerMM: renderW / drawW
            };
        }

        // 仅生成纯白画布，用于轮廓预览（不绘制原图）
        function blankFitCanvas(targetCanvas, targetCtx, containerWidth, maxHeight) {
            const drawW = Number(inputDrawW.value);
            const drawH = Number(inputDrawH.value);
            const mmRatio = drawW / drawH;
            const cW = containerWidth;
            const cH = maxHeight;
            const cRatio = cW / cH;

            let renderW, renderH;
            if (mmRatio > cRatio) {
                renderW = cW;
                renderH = cW / mmRatio;
            } else {
                renderH = cH;
                renderW = cH * mmRatio;
            }

            targetCanvas.width = Math.round(cW);
            targetCanvas.height = Math.round(cH);
            targetCtx.fillStyle = "#ffffff";
            targetCtx.fillRect(0, 0, cW, cH);
            return {
                renderW, renderH,
                offsetX: (cW - renderW) / 2,
                offsetY: (cH - renderH) / 2,
                previewPixelPerMM: renderW / drawW
            };
        }

        // 刷新原图预览
        function refreshOriginPreview() {
            if (!originImg) return;
            const containerW = cvsIn.clientWidth;
            renderFitCanvas(cvsIn, ctxIn, originImg, containerW, 340);
        }
        inputDrawW.oninput = refreshOriginPreview;
        inputDrawH.oninput = refreshOriginPreview;

        // 图片上传加载
        document.getElementById("imgFile").onchange = e => {
            const file = e.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = ev => {
                const img = new Image();
                img.onload = () => {
                    originImg = img;
                    contourList = [];
                    fullRapidText = "";
                    codeBox.value = "";
                    segInfoDom.innerText = "";
                    refreshOriginPreview();
                    // 清空轮廓预览
                    const cW = cvsOut.clientWidth;
                    blankFitCanvas(cvsOut, ctxOut, cW, 340);
                }
                img.src = ev.target.result;
            }
            reader.readAsDataURL(file);
        }

        // 道格拉斯曲线抽稀
        function distPtSeg(px, py, x1, y1, x2, y2) {
            const A = px - x1, B = py - y1, C = x2 - x1, D = y2 - y1;
            const dot = A * C + B * D, lenSq = C * C + D * D;
            let xx, yy;
            if (lenSq !== 0) {
                const t = dot / lenSq;
                xx = x1 + t * C; yy = y1 + t * D;
            } else { xx = x1; yy = y1; }
            return Math.hypot(px - xx, py - yy);
        }
        function simplifyDouglas(points, tol) {
            if (points.length <= 2) return points;
            let maxD = 0, maxI = 1;
            const p0 = points[0], pE = points.at(-1);
            for (let i = 1; i < points.length - 1; i++) {
                const d = distPtSeg(points[i].x, points[i].y, p0.x, p0.y, pE.x, pE.y);
                if (d > maxD) { maxD = d; maxI = i; }
            }
            if (maxD > tol) {
                const left = simplifyDouglas(points.slice(0, maxI + 1), tol);
                const right = simplifyDouglas(points.slice(maxI), tol);
                return left.slice(0, -1).concat(right);
            } else { return [p0, pE]; }
        }

        // 剔除相邻重复点
        function removeRepeat(points) {
            const res = [];
            for (let p of points) {
                const last = res.at(-1);
                if (!last || last.x !== p.x || last.y !== p.y) res.push(p);
            }
            return res;
        }

        // 图像边缘提取（原始像素）
        function getEdgeContours(imgW, imgH, imgData) {
            const data = imgData.data;
            const visited = new Uint8Array(imgW * imgH);
            const contours = [];
            const dir4 = [[1, 0], [-1, 0], [0, 1], [0, -1]];
            const dirTrace = [[1, 0], [0, 1], [-1, 0], [0, -1], [1, 1], [-1, 1], [-1, -1], [1, -1]];

            function isEdge(x, y) {
                const idx = y * imgW + x;
                if (data[idx * 4] > 128) return false;
                for (let [dx, dy] of dir4) {
                    const nx = x + dx, ny = y + dy;
                    if (nx < 0 || nx >= imgW || ny < 0 || ny >= imgH) return true;
                    const nIdx = ny * imgW + nx;
                    if (data[nIdx * 4] > 128) return true;
                }
                return false;
            }

            function traceContour(sx, sy) {
                const pts = [];
                let x = sx, y = sy;
                while (true) {
                    const idx = y * imgW + x;
                    if (visited[idx]) break;
                    visited[idx] = 1;
                    pts.push({ x, y });
                    let next = null;
                    for (let [dx, dy] of dirTrace) {
                        const nx = x + dx, ny = y + dy;
                        if (nx >= 0 && nx < imgW && ny >= 0 && ny < imgH) {
                            const nIdx = ny * imgW + nx;
                            if (!visited[nIdx] && isEdge(nx, ny)) {
                                next = [nx, ny]; break;
                            }
                        }
                    }
                    if (!next) break;
                    x = next[0]; y = next[1];
                }
                return pts;
            }

            for (let y = 0; y < imgH; y++) {
                for (let x = 0; x < imgW; x++) {
                    if (!visited[y * imgW + x] && isEdge(x, y)) {
                        const seg = traceContour(x, y);
                        if (seg.length > 10) contours.push(seg);
                    }
                }
            }
            return contours;
        }

        // 获取无注释纯净代码
        function getPureNoCommentCode() {
            if (!fullRapidText) return "";
            const lineArr = fullRapidText.split("\n");
            let pureCode = "";
            for (let line of lineArr) {
                const trimLine = line.trimStart();
                if (trimLine.startsWith("!")) continue;
                pureCode += line + "\n";
            }
            pureCode = pureCode.replace(/\n{3,}/g, "\n\n").trimEnd();
            return pureCode;
        }

        // 生成轨迹与绘图代码主逻辑
        document.getElementById("genTraj").onclick = () => {
            if (!originImg) { alert("请先上传图片！"); return; }
            const drawW = Number(inputDrawW.value);
            const drawH = Number(inputDrawH.value);
            const tol = Number(document.getElementById("simplifyTol").value);
            const minLen = Number(document.getElementById("minContourLen").value);
            const imgW = originImg.width;
            const imgH = originImg.height;
            const scaleX = drawW / imgW;
            const scaleY = drawH / imgH;

            // 临时画布提取原始轮廓
            const tempCanvas = document.createElement("canvas");
            const tempCtx = tempCanvas.getContext("2d");
            tempCanvas.width = imgW;
            tempCanvas.height = imgH;
            tempCtx.drawImage(originImg, 0, 0);
            const imgData = tempCtx.getImageData(0, 0, imgW, imgH);
            const rawEdgeList = getEdgeContours(imgW, imgH, imgData);
            contourList = [];
            for (let raw of rawEdgeList) {
                if (raw.length < minLen) continue;
                let clean = removeRepeat(raw);
                let smooth = simplifyDouglas(clean, tol);
                contourList.push({ raw: clean, opt: smooth });
            }

            // 轮廓预览：纯白空白画布，不绘制原图
            const containerW = cvsOut.clientWidth;
            const fitInfo = blankFitCanvas(cvsOut, ctxOut, containerW, 340);
            const previewMMScale = fitInfo.previewPixelPerMM;

            // 仅绘制彩色分段轮廓+编号
            contourList.forEach((item, idx) => {
                const pts = item.opt;
                const c = colorList[idx % colorList.length];
                ctxOut.strokeStyle = c;
                ctxOut.lineWidth = 3;
                ctxOut.beginPath();
                // 原始像素转毫米，再转预览像素，居中对齐
                const mmX0 = (pts[0].x - imgW / 2) * scaleX;
                const mmY0 = -(pts[0].y - imgH / 2) * scaleY;
                const px0 = fitInfo.offsetX + fitInfo.renderW / 2 + mmX0 * previewMMScale;
                const py0 = fitInfo.offsetY + fitInfo.renderH / 2 - mmY0 * previewMMScale;
                ctxOut.moveTo(px0, py0);
                pts.forEach(p => {
                    const mmX = (p.x - imgW / 2) * scaleX;
                    const mmY = -(p.y - imgH / 2) * scaleY;
                    const px = fitInfo.offsetX + fitInfo.renderW / 2 + mmX * previewMMScale;
                    const py = fitInfo.offsetY + fitInfo.renderH / 2 - mmY * previewMMScale;
                    ctxOut.lineTo(px, py);
                });
                ctxOut.closePath();
                ctxOut.stroke();
                // 绘制分段编号
                ctxOut.fillStyle = c;
                ctxOut.font = "bold 16px Arial";
                ctxOut.fillText(`${idx + 1}`, px0 + 4, py0 - 4);
            });

            // 生成ABB RAPID程序
            let rapid = "";
            rapid += "!====================机器人初始化固定定义====================\n";
            rapid += "TASK PERS wobjdata wobj1:=[FALSE,TRUE,\"\",[[0,0,0],[1,0,0,0]],[[0,0,0],[1,0,0,0]]];\n";
            rapid += "VAR robtarget p0:=[[0,0,0],[0,0,0,1],[0,-1,-1,0],[9E+09,9E+09,9E+09,9E+09,9E+09,9E+09]];\n\n";
            rapid += "!绘图主程序 p0为图片中心，初始移动至p0上方100mm，空走速度v1000\n";
            rapid += "PROC DrawImageTraj()\n";
            rapid += "    MoveJ Offs(p0,0,0," + safeZTop + "),v1000,fine,Tool0\\WObj:=wobj1; !移动到图片中心正上方100mm安全高度\n\n";

            contourList.forEach((item, segIdx) => {
                const pts = item.opt;
                const first = pts[0];
                const x0 = (first.x - imgW / 2) * scaleX;
                const y0 = -(first.y - imgH / 2) * scaleY;
                const zUp = penUpOffset;
                const zDown = 0;

                rapid += "!---------------------------------------------------------\n";
                rapid += `!                 【第${segIdx + 1}条平滑闭合轮廓】\n`;
                rapid += `!                 平滑后点位数量：${pts.length}\n`;
                rapid += "!---------------------------------------------------------\n";
                rapid += `    MoveL Offs(p0,${x0.toFixed(2)},${y0.toFixed(2)},${zUp}),v1000,fine,Tool0\\WObj:=wobj1; !抬笔移动至本段起点上空20mm\n`;
                rapid += `    MoveL Offs(p0,${x0.toFixed(2)},${y0.toFixed(2)},${zDown}),v500,fine,Tool0\\WObj:=wobj1; !下降20mm落笔绘图\n`;
                for (let i = 1; i < pts.length; i++) {
                    const p = pts[i];
                    const px = (p.x - imgW / 2) * scaleX;
                    const py = -(p.y - imgH / 2) * scaleY;
                    rapid += `    MoveL Offs(p0,${px.toFixed(2)},${py.toFixed(2)},${zDown}),v500,z5,Tool0\\WObj:=wobj1;\n`;
                }
                rapid += `    MoveL Offs(p0,${x0.toFixed(2)},${y0.toFixed(2)},${zDown}),v500,z5,Tool0\\WObj:=wobj1; !闭合本段轮廓\n`;
                rapid += `    MoveL Offs(p0,${x0.toFixed(2)},${y0.toFixed(2)},${zUp}),v500,fine,Tool0\\WObj:=wobj1; !本段绘制完成抬笔\n\n`;
            });

            rapid += "    MoveJ Offs(p0,0,0," + safeZTop + "),v1000,fine,Tool0\\WObj:=wobj1; !全部绘制完毕返回中心上方100mm\n";
            rapid += "ENDPROC\n\n";
            rapid += "!参数说明：\n";
            rapid += "!1. 工件原点p0位于图片画面中心\n";
            rapid += "!2. 所有空走MoveJ速度v1000；绘图MoveL v500 z5\n";
            rapid += "!3. 每段独立抬落笔20mm，仅提取图形内外边界，自动简化冗余点\n";

            fullRapidText = rapid;
            codeBox.value = fullRapidText;
        }

        // 复制带注释完整程序
        document.getElementById("copyCode").onclick = () => {
            if (!fullRapidText) { alert("请先生成轨迹代码！"); return; }
            codeBox.select();
            document.execCommand("copy");
            alert("带注释程序复制成功！");
        }

        // 复制无注释纯运行代码
        document.getElementById("copyCodeNoNote").onclick = () => {
            const pureCode = getPureNoCommentCode();
            if (!pureCode) { alert("请先生成轨迹代码！"); return; }
            const tempTextarea = document.createElement("textarea");
            tempTextarea.value = pureCode;
            document.body.appendChild(tempTextarea);
            tempTextarea.select();
            document.execCommand("copy");
            document.body.removeChild(tempTextarea);
            alert("无注释纯代码复制完成！");
        }

        // 下载标准MOD文件
        document.getElementById("downloadMod").onclick = () => {
            const pureCode = getPureNoCommentCode();
            if (!pureCode) { alert("请先生成轨迹代码！"); return; }
            const blob = new Blob([pureCode], { type: "text/plain;charset=utf-8" });
            const downloadA = document.createElement("a");
            downloadA.href = URL.createObjectURL(blob);
            downloadA.download = "ImageDrawTraj.mod";
            document.body.appendChild(downloadA);
            downloadA.click();
            URL.revokeObjectURL(downloadA.href);
            document.body.removeChild(downloadA);
            alert("MOD文件下载完成，可直接导入RobotStudio！");
        }

        // 窗口缩放重绘
        window.addEventListener("resize", () => {
            if (originImg) refreshOriginPreview();
        })
    </script>
</body>
</html>
