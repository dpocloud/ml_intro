<!--
link: ./ell_pr.css

@ell_pr
<div class="ell_pr-container">    
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
        <div class="pr-container">
            <canvas id="prCanvas@0" width="350" height="350"></canvas>
        </div>
    </div>
</div>

<script>
    // Получаем элементы DOM
    const prCanvas = document.getElementById('prCanvas@0');
    const prCtx = prCanvas.getContext('2d');
    const ellipseCanvas = document.getElementById('ellipseCanvas@0');
    const ellipseCtx = ellipseCanvas.getContext('2d');
    const thresholdSlider = document.getElementById('threshold@0');
    const thresholdValue = document.getElementById('threshold-value@0');
    const classifierType = document.getElementById('classifier-type@0');
    
    // Размеры графика PR
    const prPadding = 50;
    const prWidth = prCanvas.width - 2 * prPadding;
    const prHeight = prCanvas.height - 2 * prPadding;
    
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
    
    // Переменные для распределений данных PR
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
    
    // Вычисление точки на PR кривой для данного порога
    function calculatePrPoint(threshold) {
        const TP = positiveSamples.filter(score => score >= threshold).length;
        const FP = negativeSamples.filter(score => score >= threshold).length;
        const FN = positiveSamples.filter(score => score < threshold).length;
        const TN = negativeSamples.filter(score => score < threshold).length;
        
        const Recall = TP / (TP + FN); // Recall = Sensitivity = TPR
        let Precision = 0;
        if ((TP + FP) > 0) {
            Precision = TP / (TP + FP);
        } else {
            Precision = 1; // Если нет положительных предсказаний, Precision=1 по определению
        }
        
        return { Recall, Precision, TP, FP, FN, TN };
    }
    
    // Вычисление полной PR кривой
    function calculatePrCurve() {
        const thresholds = [1.01]; // Начинаем с порога выше 1, чтобы получить точку (0,1)
        
        // Создание отсортированного набора уникальных порогов
        const allScores = [...positiveSamples, ...negativeSamples].sort((a, b) => b - a);
        for (let i = 0; i < allScores.length; i++) {
            if (i === 0 || allScores[i] !== allScores[i-1]) {
                thresholds.push(allScores[i]);
            }
        }
        
        thresholds.push(-0.01); // Добавляем порог ниже 0, чтобы получить крайнюю точку
        
        // Вычисление точек PR кривой
        return thresholds.map(threshold => calculatePrPoint(threshold));
    }
    
    // Функция для рисования PR кривой
    function drawPrCurve(threshold) {
        // Очистка канваса
        prCtx.clearRect(0, 0, prCanvas.width, prCanvas.height);
        
        // Рисование осей
        prCtx.lineWidth = 1;
        prCtx.strokeStyle = '#000';
        prCtx.beginPath();
        
        // Ось X
        prCtx.moveTo(prPadding, prCanvas.height - prPadding);
        prCtx.lineTo(prCanvas.width - prPadding, prCanvas.height - prPadding);
        // Ось Y
        prCtx.moveTo(prPadding, prCanvas.height - prPadding);
        prCtx.lineTo(prPadding, prPadding);
        prCtx.stroke();
        
        // Метки осей
        prCtx.fillStyle = '#000';
        prCtx.font = '12px Arial';
        prCtx.textAlign = 'center';
        
        // Метки оси X (Recall)
        prCtx.fillText('Recall (Полнота)', prCanvas.width / 2, prCanvas.height - 10);
        for (let i = 0; i <= 10; i++) {
            const x = prPadding + (i / 10) * prWidth;
            prCtx.beginPath();
            prCtx.moveTo(x, prCanvas.height - prPadding);
            prCtx.lineTo(x, prCanvas.height - prPadding + 5);
            prCtx.stroke();
            prCtx.fillText(i / 10, x, prCanvas.height - prPadding + 20);
        }
        
        // Метки оси Y (Precision)
        prCtx.save();
        prCtx.translate(15, prCanvas.height / 2);
        prCtx.rotate(-Math.PI / 2);
        prCtx.fillText('Precision (Точность)', 0, 0);
        prCtx.restore();
        
        for (let i = 0; i <= 10; i++) {
            const y = prCanvas.height - prPadding - (i / 10) * prHeight;
            prCtx.beginPath();
            prCtx.moveTo(prPadding, y);
            prCtx.lineTo(prPadding - 5, y);
            prCtx.stroke();
            prCtx.textAlign = 'right';
            prCtx.fillText(i / 10, prPadding - 10, y + 4);
        }
        
        // Рисование линии случайного классификатора (горизонтальная линия)
        const selectedType = classifierType.value;
        if (selectedType === 'random') {
            // Для случайного классификатора горизонтальная линия на уровне Precision = P/(P+N)
            const P = positiveSamples.length;
            const N = negativeSamples.length;
            const randomPrecision = P / (P + N);
            
            prCtx.beginPath();
            prCtx.strokeStyle = 'grey';
            prCtx.setLineDash([5, 3]);
            const y = prCanvas.height - prPadding - randomPrecision * prHeight;
            prCtx.moveTo(prPadding, y);
            prCtx.lineTo(prCanvas.width - prPadding, y);
            prCtx.stroke();
            prCtx.setLineDash([]);
        }
        
        // Вычисление и рисование PR кривой
        const prPoints = calculatePrCurve();
        
        prCtx.beginPath();
        prCtx.strokeStyle = 'blue';
        prCtx.lineWidth = 2;

        // Сортируем точки по Recall для правильного рисования кривой
        prPoints.sort((a, b) => a.Recall - b.Recall);
        
        let firstValidPoint = true;
        
        prPoints.forEach((point, index) => {
            const x = prPadding + point.Recall * prWidth;
            const y = prCanvas.height - prPadding - point.Precision * prHeight;
            
            // Проверяем, что координаты валидны
            if (!isNaN(x) && !isNaN(y)) {
                if (firstValidPoint) {
                    prCtx.moveTo(x, y);
                    firstValidPoint = false;
                } else {
                    prCtx.lineTo(x, y);
                }
            }
        });
        
        prCtx.stroke();
        
        // Рисование текущей точки на PR кривой для выбранного порога
        const currentPoint = calculatePrPoint(threshold);
        
        const x = prPadding + currentPoint.Recall * prWidth;
        const y = prCanvas.height - prPadding - currentPoint.Precision * prHeight;
        
        prCtx.beginPath();
        prCtx.fillStyle = 'red';
        prCtx.arc(x, y, 6, 0, 2 * Math.PI);
        prCtx.fill();
        
        // Вычисление и отображение AP (Average Precision - площади под PR кривой)
        const ap = calculateAP(prPoints);
        prCtx.fillStyle = '#000';
        prCtx.textAlign = 'right';
        prCtx.fillText(`AP: ${ap.toFixed(3)}`, prCanvas.width - prPadding, prPadding - 10);
        
        // Отображение текущих координат точки
        prCtx.fillText(`Текущая точка: (${currentPoint.Recall.toFixed(2)}, ${currentPoint.Precision.toFixed(2)})`, 
                    prCanvas.width - prPadding, prPadding - 30);
        
        // Отображение текущих Recall и Precision значений
        prCtx.textAlign = 'left';
        prCtx.fillText(`Recall: ${currentPoint.Recall.toFixed(3)}`, prPadding + 10, prPadding - 10);
        prCtx.fillText(`Precision: ${currentPoint.Precision.toFixed(3)}`, prPadding + 10, prPadding - 30);
        
        // Показать текущие значения TP, FP, TN, FN
        prCtx.textAlign = 'left';
        prCtx.fillText(`TP: ${currentPoint.TP}`, prPadding + 10, prPadding - 50);
        prCtx.fillText(`FP: ${currentPoint.FP}`, prPadding + 10, prPadding - 70);
        prCtx.fillText(`TN: ${currentPoint.TN}`, prPadding + 120, prPadding - 50);
        prCtx.fillText(`FN: ${currentPoint.FN}`, prPadding + 120, prPadding - 70);
        
        return currentPoint;
    }
    
    // Вычисление Average Precision (AP) - площади под PR кривой
    function calculateAP(prPoints) {
        // Отсортируем точки по Recall для корректного расчета
        prPoints.sort((a, b) => a.Recall - b.Recall);
        
        let ap = 0;
        
        // Используем метод трапеций для аппроксимации площади под кривой
        for (let i = 1; i < prPoints.length; i++) {
            // Ширина трапеции
            const deltaRecall = prPoints[i].Recall - prPoints[i-1].Recall;
            // Средняя высота трапеции
            const avgPrecision = (prPoints[i].Precision + prPoints[i-1].Precision) / 2;
            
            // Площадь трапеции
            if (deltaRecall > 0) {  // Чтобы избежать ошибок при одинаковых Recall
                ap += deltaRecall * avgPrecision;
            }
        }
        
        return ap;
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
        else if (selectedType == 'perfect' && normalizedThreshold < 0.25) {
            addValue = config.sliderMin + normalizedThreshold * (0 - config.sliderMin);
        }
        else if (selectedType == 'perfect' && normalizedThreshold > 0.75) {
            addValue = 0 + normalizedThreshold * (config.sliderMax - 0);
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
        
        // Обновление PR-кривой
        const currentPoint = drawPrCurve(threshold);
        
        // Обновление эллипса
        drawEllipse(threshold);
    }
    
    // Инициализация визуализации
    updateThreshold();
</script>
@end
-->

# ell_pr

@ell_pr(0)