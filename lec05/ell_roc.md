<!--
link: ./ell_roc.css

@ell_roc

<div class="ell_roc-container">    
    <div class="controls">
        <div class="classifier-selector">
            <label for="classifier-type@0">Выберите тип классификатора:</label>
            <select id="classifier-type@0">
                <option value="real">Реальный классификатор</option>
                <option value="random">Случайный классификатор</option>
                <option value="perfect">Идеальный классификатор</option>
            </select>
        </div>
        
        <div class="slider-container">
            <label for="threshold@0">Порог (b):</label>
            <input type="range" id="threshold@0" min="0" max="1" step="0.01" value="0.5">
            <span id="threshold-value@0" class="slider-value">0.5</span>
        </div>
    </div>
    
    <div class="visualization-container">
        <div class="ellipse-container">
            <canvas id="ellipseCanvas@0" width="350" height="350"></canvas>
        </div>
        <div class="roc-container">
            <canvas id="rocCanvas@0" width="350" height="350"></canvas>
        </div>
    </div>
</div>

<script>
    // Получаем элементы DOM
    const rocCanvas = document.getElementById('rocCanvas@0');
    const rocCtx = rocCanvas.getContext('2d');
    const ellipseCanvas = document.getElementById('ellipseCanvas@0');
    const ellipseCtx = ellipseCanvas.getContext('2d');
    const thresholdSlider = document.getElementById('threshold@0');
    const thresholdValue = document.getElementById('threshold-value@0');
    const classifierType = document.getElementById('classifier-type@0');
    
    // Размеры графика ROC
    const rocPadding = 50;
    const rocWidth = rocCanvas.width - 2 * rocPadding;
    const rocHeight = rocCanvas.height - 2 * rocPadding;
    
    // Конфигурация для эллипса
    const ellipseConfig = {
        real: { angleDeg: 60, sliderMin: -2.4, sliderMax: 2.4, sliderStep: 0.1 },
        random: { angleDeg: 89.99, sliderMin: -7500, sliderMax: 7500, sliderStep: 1000 },
        perfect: { angleDeg: 0, sliderMin: -0.8, sliderMax: 0.8, sliderStep: 0.1 },
        // Цвета
        tpColor: [0, 125, 187],
        fnColor: [255, 170, 79],
        fpColor: [158, 219, 235],
        tnColor: [235, 0, 0],
        a: 1.3,        // ellipse width
        bEllipse: 0.8, // ellipse height
        canvasSize: 350,
        plotRange: 1.5
    };
    
    // Переменные для распределений данных ROC
    let positiveSamples = [];
    let negativeSamples = [];
    
    // Инициализация данных при загрузке страницы
    initializeData();
    
    // Добавляем обработчики событий
    thresholdSlider.addEventListener('input', updateThreshold);
    classifierType.addEventListener('change', function() {
        initializeData();
        updateThreshold();
    });
    
    // Функция инициализации данных
    function initializeData() {
        const selectedType = classifierType.value;
        
        if (selectedType === 'perfect') {
            // Идеальный классификатор: положительные и отрицательные примеры полностью разделены
            positiveSamples = Array.from({length: 100}, () => 0.75 + Math.random() * 0.25);
            negativeSamples = Array.from({length: 100}, () => Math.random() * 0.25);
        } else if (selectedType === 'random') {
            // Случайный классификатор: положительные и отрицательные примеры имеют одинаковое распределение
            positiveSamples = Array.from({length: 100}, () => Math.random());
            negativeSamples = Array.from({length: 100}, () => Math.random());
        } else { // 'real'
            // Реальный классификатор: положительные и отрицательные примеры перекрываются, но имеют разные средние
            positiveSamples = Array.from({length: 100}, () => 
                Math.min(1, Math.max(0, 0.65 + 0.2 * randn())));
            negativeSamples = Array.from({length: 100}, () => 
                Math.min(1, Math.max(0, 0.35 + 0.2 * randn())));
        }
    }
    
    // Функция для генерации нормально распределенных случайных чисел
    function randn() {
        let u = 0, v = 0;
        while (u === 0) u = Math.random();
        while (v === 0) v = Math.random();
        return Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2.0 * Math.PI * v);
    }
    
    // Вычисление точки на ROC кривой для данного порога
    function calculateRocPoint(threshold) {
        const TP = positiveSamples.filter(score => score >= threshold).length;
        const FP = negativeSamples.filter(score => score >= threshold).length;
        const FN = positiveSamples.filter(score => score < threshold).length;
        const TN = negativeSamples.filter(score => score < threshold).length;
        
        const TPR = TP / (TP + FN); // sensitivity
        const FPR = FP / (FP + TN); // 1 - specificity
        
        return { FPR, TPR, TP, FP, FN, TN };
    }
    
    // Вычисление полной ROC кривой
    function calculateRocCurve() {
        const thresholds = [1.01]; // Начинаем с порога выше 1, чтобы получить точку (0,0)
        
        // Создание отсортированного набора уникальных порогов
        const allScores = [...positiveSamples, ...negativeSamples].sort((a, b) => b - a);
        for (let i = 0; i < allScores.length; i++) {
            if (i === 0 || allScores[i] !== allScores[i-1]) {
                thresholds.push(allScores[i]);
            }
        }
        
        thresholds.push(-0.01); // Добавляем порог ниже 0, чтобы получить точку (1,1)
        
        // Вычисление точек ROC кривой
        return thresholds.map(threshold => calculateRocPoint(threshold));
    }
    
    // Функция для рисования ROC кривой
    function drawRocCurve(threshold) {
        // Очистка канваса
        rocCtx.clearRect(0, 0, rocCanvas.width, rocCanvas.height);
        
        // Рисование осей
        rocCtx.lineWidth = 1;
        rocCtx.strokeStyle = '#000';
        rocCtx.beginPath();
        
        // Ось X
        rocCtx.moveTo(rocPadding, rocCanvas.height - rocPadding);
        rocCtx.lineTo(rocCanvas.width - rocPadding, rocCanvas.height - rocPadding);
        // Ось Y
        rocCtx.moveTo(rocPadding, rocCanvas.height - rocPadding);
        rocCtx.lineTo(rocPadding, rocPadding);
        rocCtx.stroke();
        
        // Метки осей
        rocCtx.fillStyle = '#000';
        rocCtx.font = '12px Arial';
        rocCtx.textAlign = 'center';
        
        // Метки оси X
        rocCtx.fillText('1 - Специфичность (FPR)', rocCanvas.width / 2, rocCanvas.height - 10);
        for (let i = 0; i <= 10; i++) {
            const x = rocPadding + (i / 10) * rocWidth;
            rocCtx.beginPath();
            rocCtx.moveTo(x, rocCanvas.height - rocPadding);
            rocCtx.lineTo(x, rocCanvas.height - rocPadding + 5);
            rocCtx.stroke();
            rocCtx.fillText(i / 10, x, rocCanvas.height - rocPadding + 20);
        }
        
        // Метки оси Y
        rocCtx.save();
        rocCtx.translate(15, rocCanvas.height / 2);
        rocCtx.rotate(-Math.PI / 2);
        rocCtx.fillText('Чувствительность (TPR)', 0, 0);
        rocCtx.restore();
        
        for (let i = 0; i <= 10; i++) {
            const y = rocCanvas.height - rocPadding - (i / 10) * rocHeight;
            rocCtx.beginPath();
            rocCtx.moveTo(rocPadding, y);
            rocCtx.lineTo(rocPadding - 5, y);
            rocCtx.stroke();
            rocCtx.textAlign = 'right';
            rocCtx.fillText(i / 10, rocPadding - 10, y + 4);
        }
        
        // Рисование линии случайного классификатора (диагональ)
        rocCtx.beginPath();
        rocCtx.strokeStyle = 'grey';
        rocCtx.setLineDash([5, 3]);
        rocCtx.moveTo(rocPadding, rocCanvas.height - rocPadding);
        rocCtx.lineTo(rocCanvas.width - rocPadding, rocPadding);
        rocCtx.stroke();
        rocCtx.setLineDash([]);
        
        // Вычисление и рисование ROC кривой
        const rocPoints = calculateRocCurve();
        
        rocCtx.beginPath();
        rocCtx.strokeStyle = 'blue';
        rocCtx.lineWidth = 2;
        
        rocPoints.forEach((point, index) => {
            const x = rocPadding + point.FPR * rocWidth;
            const y = rocCanvas.height - rocPadding - point.TPR * rocHeight;
            
            if (index === 0) {
                rocCtx.moveTo(x, y);
            } else {
                rocCtx.lineTo(x, y);
            }
        });
        
        rocCtx.stroke();
        
        // Рисование текущей точки на ROC кривой для выбранного порога
        const currentPoint = calculateRocPoint(threshold);
        
        const x = rocPadding + currentPoint.FPR * rocWidth;
        const y = rocCanvas.height - rocPadding - currentPoint.TPR * rocHeight;
        
        rocCtx.beginPath();
        rocCtx.fillStyle = 'red';
        rocCtx.arc(x, y, 6, 0, 2 * Math.PI);
        rocCtx.fill();
        
        // Вычисление и отображение AUC (площади под кривой)
        const auc = calculateAUC(rocPoints);
        rocCtx.fillStyle = '#000';
        rocCtx.textAlign = 'right';
        rocCtx.fillText(`AUC: ${auc.toFixed(3)}`, rocCanvas.width - rocPadding, rocPadding - 10);
        
        // Отображение текущих координат точки
        rocCtx.fillText(`Текущая точка: (${currentPoint.FPR.toFixed(2)}, ${currentPoint.TPR.toFixed(2)})`, 
                    rocCanvas.width - rocPadding, rocPadding - 30);
        
        // Отображение текущих TPR и FPR значений
        rocCtx.textAlign = 'left';
        rocCtx.fillText(`TPR: ${currentPoint.TPR.toFixed(3)}`, rocPadding + 10, rocPadding - 10);
        rocCtx.fillText(`FPR: ${currentPoint.FPR.toFixed(3)}`, rocPadding + 10, rocPadding - 30);
        
        // Показать текущие значения TP, FP, TN, FN
        rocCtx.textAlign = 'left';
        rocCtx.fillText(`TP: ${currentPoint.TP}`, rocPadding + 10, rocPadding - 50);
        rocCtx.fillText(`FP: ${currentPoint.FP}`, rocPadding + 10, rocPadding - 70);
        rocCtx.fillText(`TN: ${currentPoint.TN}`, rocPadding + 120, rocPadding - 50);
        rocCtx.fillText(`FN: ${currentPoint.FN}`, rocPadding + 120, rocPadding - 70);
        
        return currentPoint;
    }
    
    // Вычисление площади под ROC кривой (AUC)
    function calculateAUC(rocPoints) {
        let auc = 0;
        for (let i = 1; i < rocPoints.length; i++) {
            // Метод трапеций для вычисления AUC
            const width = rocPoints[i].FPR - rocPoints[i-1].FPR;
            const height = (rocPoints[i].TPR + rocPoints[i-1].TPR) / 2;
            auc += width * height;
        }
        return auc;
    }
    
    // Функция для рисования эллипса с разделяющей линией
    function drawEllipse(normalizedThreshold) {
        const selectedType = classifierType.value;
        const config = ellipseConfig[selectedType];
        const canvasSize = ellipseConfig.canvasSize;
        
        // Преобразование нормализованного порога (0-1) в значение для эллипса
        let addValue;
        if (selectedType == 'perfect' && normalizedThreshold >= 0.25 && normalizedThreshold <= 0.75) {
            addValue = 0;
        }
        else {
            addValue = config.sliderMin + normalizedThreshold * (config.sliderMax - config.sliderMin);
        }
        const add = addValue;
        
        // Очистка canvas
        ellipseCtx.clearRect(0, 0, canvasSize, canvasSize);
        
        // Масштаб для преобразования координат данных в пиксели
        const scale = canvasSize / (2 * ellipseConfig.plotRange);
        
        // Вычисление параметров
        const theta = config.angleDeg * Math.PI / 180;
        const slope = Math.tan(theta);
        
        // Создаем внеэкранный canvas для манипуляции пикселями
        const pixelCanvas = document.createElement('canvas');
        pixelCanvas.width = canvasSize;
        pixelCanvas.height = canvasSize;
        const pixelCtx = pixelCanvas.getContext('2d');
        const imageData = pixelCtx.createImageData(canvasSize, canvasSize);
        const data = imageData.data;
        
        // Функции для преобразования координат
        function toPixelX(x) {
            return (x + ellipseConfig.plotRange) * scale;
        }

        function toPixelY(y) {
            return canvasSize - (y + ellipseConfig.plotRange) * scale;
        }
        
        // Выборка точек и раскрашивание секторов
        for (let i = 0; i < canvasSize; i++) {
            for (let j = 0; j < canvasSize; j++) {
                // Преобразование пикселя в координаты данных
                const x = (i / scale) - ellipseConfig.plotRange;
                const y = ellipseConfig.plotRange - (j / scale);
                
                // Проверка, находится ли точка внутри эллипса
                const inEllipse = (x*x)/(ellipseConfig.a*ellipseConfig.a) + (y*y)/(ellipseConfig.bEllipse*ellipseConfig.bEllipse) <= 1;
                
                if (inEllipse) {
                    const idx = (j * canvasSize + i) * 4;
                    
                    // Определение сектора
                    if (y < 0 && y > slope * x + add) {
                        // FP сектор
                        data[idx] = ellipseConfig.fpColor[0];
                        data[idx+1] = ellipseConfig.fpColor[1];
                        data[idx+2] = ellipseConfig.fpColor[2];
                    } else if (y > 0 && y > slope * x + add) {
                        // TP сектор
                        data[idx] = ellipseConfig.tpColor[0];
                        data[idx+1] = ellipseConfig.tpColor[1];
                        data[idx+2] = ellipseConfig.tpColor[2];
                    } else if (y < 0 && y < slope * x + add) {
                        // TN сектор
                        data[idx] = ellipseConfig.tnColor[0];
                        data[idx+1] = ellipseConfig.tnColor[1];
                        data[idx+2] = ellipseConfig.tnColor[2];
                    } else if (y > 0 && y < slope * x + add) {
                        // FN сектор
                        data[idx] = ellipseConfig.fnColor[0];
                        data[idx+1] = ellipseConfig.fnColor[1];
                        data[idx+2] = ellipseConfig.fnColor[2];
                    }
                    data[idx+3] = 255; // Alpha channel
                }
            }
        }
        
        // Возвращаем пиксельные данные обратно на внеэкранный canvas
        pixelCtx.putImageData(imageData, 0, 0);
        
        // Рисуем внеэкранный canvas на основной canvas
        ellipseCtx.drawImage(pixelCanvas, 0, 0);
        
        // Рисуем контур эллипса
        ellipseCtx.beginPath();
        ellipseCtx.ellipse(
            toPixelX(0), 
            toPixelY(0), 
            ellipseConfig.a * scale, 
            ellipseConfig.bEllipse * scale, 
            0, 0, 2 * Math.PI
        );
        ellipseCtx.strokeStyle = 'white';
        ellipseCtx.lineWidth = 2;
        ellipseCtx.stroke();
        
        // Рисуем разделяющую линию
        ellipseCtx.beginPath();
        const x1 = -ellipseConfig.plotRange;
        const y1 = slope * x1 + add;
        const x2 = ellipseConfig.plotRange;
        const y2 = slope * x2 + add;
        ellipseCtx.moveTo(toPixelX(x1), toPixelY(y1));
        ellipseCtx.lineTo(toPixelX(x2), toPixelY(y2));
        ellipseCtx.strokeStyle = 'black';
        ellipseCtx.lineWidth = 3;
        ellipseCtx.stroke();
        
        // Добавляем метки секторов
        function calculateSectorCenter(sectorType) {
            let xSum = 0, ySum = 0, count = 0;
            const step = 0.02;
            
            for (let x = -ellipseConfig.a; x <= ellipseConfig.a; x += step) {
                for (let y = -ellipseConfig.bEllipse; y <= ellipseConfig.bEllipse; y += step) {
                    if ((x*x)/(ellipseConfig.a*ellipseConfig.a) + (y*y)/(ellipseConfig.bEllipse*ellipseConfig.bEllipse) <= 1) {
                        let isInSector = false;
                        
                        if (sectorType === 'TP' && y > 0 && y > slope * x + add) {
                            isInSector = true;
                        } else if (sectorType === 'FN' && y > 0 && y < slope * x + add) {
                            isInSector = true;
                        } else if (sectorType === 'FP' && y < 0 && y > slope * x + add) {
                            isInSector = true;
                        } else if (sectorType === 'TN' && y < 0 && y < slope * x + add) {
                            isInSector = true;
                        }
                        
                        if (isInSector) {
                            xSum += x;
                            ySum += y;
                            count++;
                        }
                    }
                }
            }
            
            if (count > 0) {
                return [xSum / count, ySum / count];
            }
            return null;
        }
        
        // Добавляем метки для каждого сектора
        const tpCenter = calculateSectorCenter('TP');
        const fnCenter = calculateSectorCenter('FN');
        const fpCenter = calculateSectorCenter('FP');
        const tnCenter = calculateSectorCenter('TN');
        
        ellipseCtx.font = 'bold 20px Arial';
        ellipseCtx.textAlign = 'center';
        ellipseCtx.textBaseline = 'middle';
        ellipseCtx.fillStyle = 'white';
        
        if (tpCenter) ellipseCtx.fillText("TP", toPixelX(tpCenter[0]), toPixelY(tpCenter[1]));
        if (fnCenter) ellipseCtx.fillText("FN", toPixelX(fnCenter[0]), toPixelY(fnCenter[1]));
        if (fpCenter) ellipseCtx.fillText("FP", toPixelX(fpCenter[0]), toPixelY(fpCenter[1]));
        if (tnCenter) ellipseCtx.fillText("TN", toPixelX(tnCenter[0]), toPixelY(tnCenter[1]));
    }
    
    // Обновление визуализации при изменении порога
    function updateThreshold() {
        const threshold = parseFloat(thresholdSlider.value);
        thresholdValue.textContent = threshold.toFixed(2);
        
        // Обновление ROC-кривой
        const currentPoint = drawRocCurve(threshold);
        
        // Обновление эллипса
        drawEllipse(threshold);
    }
    
    // Инициализация визуализации
    updateThreshold();
</script>
@end
-->

# ell_roc

@ell_roc(0)
@ell_roc(1)