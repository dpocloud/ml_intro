<!--
link: ./window.css

@window
<div class="window-container">
    <div class="controls">
        <div class="control-group">
            <label for="windowPos@0">Положение окна (x):</label>
            <input type="range" id="windowPos@0" min="0" max="10" value="5" step="0.1">
            <div class="value-display" id="windowPosValue@0">5.0</div>
        </div>
        <div class="control-group">
            <label for="windowWidth@0">Ширина окна (h):</label>
            <input type="range" id="windowWidth@0" min="0.5" max="3" value="1.5" step="0.1">
            <div class="value-display" id="windowWidthValue@0">1.5</div>
        </div>
        <div class="control-group">
            <button onclick="window.generateNewData_@0()" class="control-group button">Сгенерировать данные</button>
        </div>
    </div>
    <div class="formula">
        <span id="currentFormula@0">f(5.0) =3 / (20 × 1.5) = 0.100</span>
    </div>
    <div class="visualization">
        <svg id="mainSvg@0" viewBox="0 0 800 300">
            <g id="dataVisualization@0" transform="translate(80, 50)">

                <line class="axis" x1="0" y1="200" x2="640" y2="200"></line>
                <line class="axis" x1="0" y1="0" x2="0" y2="200"></line>

                <text class="axis-label" x="320" y="235">x</text>
                <text class="axis-label" x="-25" y="100" transform="rotate(-90, -25, 100)">f(x)</text>
                
                <rect id="slidingWindow@0" class="window" x="0" y="180" width="96" height="20"></rect>
                
                <g id="dataPoints@0"></g>
                
                <path id="densityCurve@0" class="density-curve"></path>
                
                <g id="xAxisLabels@0"></g>
                
                <g id="yAxisLabels@0"></g>
            </g>
        </svg>
    </div>
</div>

<script>
(function() {
    // Данные и параметры
    let dataPoints = [];
    let windowPosition = 5.0;
    let windowWidth = 1.5;
    const totalRange = 10;
    const svgWidth = 640;
    const svgHeight = 200;
    
    // Ждем, пока DOM полностью загрузится
    function waitForElement(selector, callback) {
        const element = document.getElementById(selector);
        if (element) {
            callback();
        } else {
            setTimeout(() => waitForElement(selector, callback), 100);
        }
    }
    
    // Инициализация
    function init() {
        generateNewData();
        setupEventListeners();
        updateVisualization();
    }
    
    // Генерация новых данных
    function generateNewData() {
        dataPoints = [];
        
        // Генерируем кластеры точек для более интересного распределения
        const clusters = [
            { center: 2, spread: 0.8, count: 6 },
            { center: 5, spread: 1.2, count: 8 },
            { center: 8, spread: 0.6, count: 6 }
        ];
        
        clusters.forEach(cluster => {
            for (let i = 0; i < cluster.count; i++) {
                const point = cluster.center + (Math.random() - 0.5) * cluster.spread * 2;
                if (point >= 0 && point <= totalRange) {
                    dataPoints.push(Math.max(0, Math.min(totalRange, point)));
                }
            }
        });
        
        dataPoints.sort((a, b) => a - b);
        
        const totalElement = document.getElementById('totalPoints@0');
        if (totalElement) {
            totalElement.textContent = dataPoints.length;
        }
        
        updateVisualization();
    }
    
    // Настройка обработчиков событий
    function setupEventListeners() {
        const windowPosSlider = document.getElementById('windowPos@0');
        const windowWidthSlider = document.getElementById('windowWidth@0');
        
        if (windowPosSlider) {
            windowPosSlider.addEventListener('input', (e) => {
                windowPosition = parseFloat(e.target.value);
                const valueElement = document.getElementById('windowPosValue@0');
                if (valueElement) {
                    valueElement.textContent = windowPosition.toFixed(1);
                }
                updateVisualization();
            });
        }
        
        if (windowWidthSlider) {
            windowWidthSlider.addEventListener('input', (e) => {
                windowWidth = parseFloat(e.target.value);
                const valueElement = document.getElementById('windowWidthValue@0');
                if (valueElement) {
                    valueElement.textContent = windowWidth.toFixed(1);
                }
                updateVisualization();
            });
        }
    }
    
    // Преобразование координат
    function xToSvg(x) {
        return (x / totalRange) * svgWidth;
    }
    
    function yToSvg(y, maxY) {
        return svgHeight - (y / maxY) * (svgHeight - 20);
    }
    
    // Вычисление плотности в точке
    function calculateDensity(x) {
        const leftBound = x - windowWidth / 2;
        const rightBound = x + windowWidth / 2;
        
        const pointsInWindow = dataPoints.filter(point => 
            point >= leftBound && point <= rightBound
        ).length;
        
        return pointsInWindow / (dataPoints.length * windowWidth);
    }
    
    // Подсчет точек в текущем окне
    function getPointsInCurrentWindow() {
        const leftBound = windowPosition - windowWidth / 2;
        const rightBound = windowPosition + windowWidth / 2;
        
        return dataPoints.filter(point => 
            point >= leftBound && point <= rightBound
        );
    }
    
    // Обновление визуализации
    function updateVisualization() {
        updateDataPoints();
        updateWindow();
        updateDensityCurve();
        updateInfo();
        updateAxes();
    }
    
    // Обновление точек данных
    function updateDataPoints() {
        const pointsGroup = document.getElementById('dataPoints@0');
        if (!pointsGroup) return;
        
        pointsGroup.innerHTML = '';
        
        const pointsInWindow = getPointsInCurrentWindow();
        
        dataPoints.forEach(point => {
            const circle = document.createElementNS('http://www.w3.org/2000/svg', 'circle');
            circle.setAttribute('cx', xToSvg(point));
            circle.setAttribute('cy', 190);
            circle.setAttribute('r', 4);
            
            if (pointsInWindow.includes(point)) {
                circle.setAttribute('class', 'data-point in-window');
            } else {
                circle.setAttribute('class', 'data-point');
            }
            
            pointsGroup.appendChild(circle);
        });
    }
    
    // Обновление окна
    function updateWindow() {
        const window = document.getElementById('slidingWindow@0');
        if (!window) return;
        
        const leftBound = windowPosition - windowWidth / 2;
        const rightBound = windowPosition + windowWidth / 2;
        
        window.setAttribute('x', xToSvg(Math.max(0, leftBound)));
        window.setAttribute('width', xToSvg(Math.min(totalRange, rightBound)) - xToSvg(Math.max(0, leftBound)));
    }
    
    // Обновление кривой плотности
    function updateDensityCurve() {
        const curve = document.getElementById('densityCurve@0');
        if (!curve) return;
        
        const step = 0.1;
        let pathData = '';
        let maxDensity = 0;
        
        // Находим максимальную плотность для масштабирования
        for (let x = 0; x <= totalRange; x += step) {
            const density = calculateDensity(x);
            maxDensity = Math.max(maxDensity, density);
        }
        
        // Строим кривую
        for (let x = 0; x <= totalRange; x += step) {
            const density = calculateDensity(x);
            const svgX = xToSvg(x);
            const svgY = yToSvg(density, maxDensity);
            
            if (x === 0) {
                pathData += `M ${svgX} ${svgY}`;
            } else {
                pathData += ` L ${svgX} ${svgY}`;
            }
        }
        
        curve.setAttribute('d', pathData);
        
        const maxDensityElement = document.getElementById('maxDensity@0');
        if (maxDensityElement) {
            maxDensityElement.textContent = maxDensity.toFixed(3);
        }
    }
    
    // Обновление информационной панели
    function updateInfo() {
        const pointsInWindow = getPointsInCurrentWindow();
        const currentDensity = calculateDensity(windowPosition);
        
        const pointsInWindowElement = document.getElementById('pointsInWindow@0');
        if (pointsInWindowElement) {
            pointsInWindowElement.textContent = pointsInWindow.length;
        }
        
        const currentDensityElement = document.getElementById('currentDensity@0');
        if (currentDensityElement) {
            currentDensityElement.textContent = currentDensity.toFixed(3);
        }
        
        // Обновление формулы
        const formula = `f(${windowPosition.toFixed(1)}) = ${pointsInWindow.length} / (${dataPoints.length} × ${windowWidth.toFixed(1)}) = ${currentDensity.toFixed(3)}`;
        const formulaElement = document.getElementById('currentFormula@0');
        if (formulaElement) {
            formulaElement.textContent = formula;
        }
    }
    
    // Обновление осей
    function updateAxes() {
        const xLabelsGroup = document.getElementById('xAxisLabels@0');
        if (!xLabelsGroup) return;
        
        xLabelsGroup.innerHTML = '';
        
        for (let i = 0; i <= totalRange; i += 2) {
            const line = document.createElementNS('http://www.w3.org/2000/svg', 'line');
            line.setAttribute('x1', xToSvg(i));
            line.setAttribute('x2', xToSvg(i));
            line.setAttribute('y1', 200);
            line.setAttribute('y2', 205);
            line.setAttribute('stroke', '#2c3e50');
            line.setAttribute('stroke-width', 1);
            xLabelsGroup.appendChild(line);
            
            const text = document.createElementNS('http://www.w3.org/2000/svg', 'text');
            text.setAttribute('x', xToSvg(i));
            text.setAttribute('y', 220);
            text.setAttribute('class', 'axis-label');
            text.setAttribute('font-size', '12');
            text.textContent = i;
            xLabelsGroup.appendChild(text);
        }
    }
    
    // Экспортируем функцию для кнопки
    window.generateNewData_@0 = generateNewData;
    
    // Запуск приложения с задержкой
    waitForElement('windowPos@0', init);
})();
</script>
@end
-->

# window

@window(0)

@window(1)