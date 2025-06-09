<!--
link: ./conf_matrix.css

@conf_matrix

<div class="cm-container">
    <h1>Multiclass Confusion Matrix</h1>
    <p class="subtitle">One-vs-All Approach</p>
    
    <div class="matrices-container">
        <div class="matrix-section">
            <div class="matrix-title">Multiclass Confusion Matrix (5×5)</div>
            <div id="multiclassMatrix@0" class="matrix multiclass-matrix"></div>
        </div>

        <div class="matrix-section">
            <div class="matrix-title">Binary Confusion Matrix</div>
            <div id="binaryMatrix@0" class="matrix binary-matrix"></div>
            <div class="controls">
                <label for="classSelect@0">Выберите позитивный класс: </label>
                <select id="classSelect@0" class="class-selector">
                    <option value="0">Class 1</option>
                    <option value="1">Class 2</option>
                    <option value="2">Class 3</option>
                    <option value="3">Class 4</option>
                    <option value="4">Class 5</option>
                </select>
            </div>
        </div>
    </div>
</div>

<script>
    // Пример данных для многоклассовой матрицы ошибок 5×5
    const confusionMatrix = [
        [85, 2, 3, 1, 4],
        [3, 78, 5, 2, 2],
        [1, 4, 82, 3, 0],
        [2, 1, 2, 88, 2],
        [0, 3, 1, 1, 89]
    ];

    const classNames = ['Class 1', 'Class 2', 'Class 3', 'Class 4', 'Class 5'];
    let currentClass = 0;

    function createMulticlassMatrix() {
        const container = document.getElementById('multiclassMatrix@0');
        container.innerHTML = '';

        // Заголовок "Prediction"
        const predictionHeader = document.createElement('div');
        predictionHeader.className = 'cell header-cell';
        predictionHeader.style.gridColumn = '3 / -1';
        predictionHeader.style.gridRow = '1';
        predictionHeader.textContent = 'Prediction';
        container.appendChild(predictionHeader);

        // Заголовки столбцов классов
        for (let i = 0; i < 5; i++) {
            const header = document.createElement('div');
            header.className = 'cell header-cell class-header';
            header.style.gridColumn = i + 3;
            header.style.gridRow = '2';
            header.textContent = classNames[i];
            container.appendChild(header);
        }

        // Заголовок "Actual"
        const actualHeader = document.createElement('div');
        actualHeader.className = 'cell header-cell';
        actualHeader.style.gridColumn = '1';
        actualHeader.style.gridRow = '3 / -1';
        actualHeader.style.writingMode = 'vertical-rl';
        actualHeader.style.textOrientation = 'mixed';
        actualHeader.textContent = 'Actual';
        container.appendChild(actualHeader);

        // Заголовки строк классов
        for (let i = 0; i < 5; i++) {
            const header = document.createElement('div');
            header.className = 'cell header-cell class-header';
            header.style.gridColumn = '2';
            header.style.gridRow = i + 3;
            header.textContent = classNames[i];
            container.appendChild(header);
        }

        // Данные матрицы
        for (let i = 0; i < 5; i++) {
            for (let j = 0; j < 5; j++) {
                const cell = document.createElement('div');
                cell.className = 'cell data-cell';
                // Выделяем зеленым только диагональный элемент текущего выбранного класса
                if (i === j && i === currentClass) {
                    cell.classList.add('current-class-diagonal');
                }
                cell.style.gridColumn = j + 3;
                cell.style.gridRow = i + 3;
                cell.textContent = confusionMatrix[i][j];
                cell.dataset.row = i;
                cell.dataset.col = j;
                container.appendChild(cell);
            }
        }
    }

    function createBinaryMatrix() {
        const container = document.getElementById('binaryMatrix@0');
        container.innerHTML = '';

        // Заголовок "Prediction"
        const predictionHeader = document.createElement('div');
        predictionHeader.className = 'cell header-cell';
        predictionHeader.style.gridColumn = '4 / -1';
        predictionHeader.style.gridRow = '1';
        predictionHeader.textContent = 'Prediction';
        container.appendChild(predictionHeader);

        // Заголовки столбцов
        const posHeader = document.createElement('div');
        posHeader.className = 'cell header-cell';
        posHeader.style.gridColumn = '4';
        posHeader.style.gridRow = '2';
        posHeader.textContent = 'Positive';
        container.appendChild(posHeader);

        const negHeader = document.createElement('div');
        negHeader.className = 'cell header-cell';
        negHeader.style.gridColumn = '5';
        negHeader.style.gridRow = '2';
        negHeader.textContent = 'Negative';
        container.appendChild(negHeader);

        // Заголовок "Actual"
        const actualHeader = document.createElement('div');
        actualHeader.className = 'cell header-cell';
        actualHeader.style.gridColumn = '1';
        actualHeader.style.gridRow = '3 / -1';
        actualHeader.style.writingMode = 'vertical-rl';
        actualHeader.style.textOrientation = 'mixed';
        actualHeader.textContent = 'Actual';
        container.appendChild(actualHeader);

        // Заголовки строк
        const posRowHeader = document.createElement('div');
        posRowHeader.className = 'cell header-cell';
        posRowHeader.style.gridColumn = '3';
        posRowHeader.style.gridRow = '3';
        posRowHeader.textContent = 'Positive';
        container.appendChild(posRowHeader);

        const negRowHeader = document.createElement('div');
        negRowHeader.className = 'cell header-cell';
        negRowHeader.style.gridColumn = '3';
        negRowHeader.style.gridRow = '4';
        negRowHeader.textContent = 'Negative';
        container.appendChild(negRowHeader);

        // Вычисляем значения для бинарной матрицы
        const tp = confusionMatrix[currentClass][currentClass];
        let fp = 0, fn = 0, tn = 0;

        // FP: сумма столбца без диагонального элемента
        for (let i = 0; i < 5; i++) {
            if (i !== currentClass) {
                fp += confusionMatrix[i][currentClass];
            }
        }

        // FN: сумма строки без диагонального элемента
        for (let j = 0; j < 5; j++) {
            if (j !== currentClass) {
                fn += confusionMatrix[currentClass][j];
            }
        }

        // TN: сумма всех элементов не в строке и не в столбце
        for (let i = 0; i < 5; i++) {
            for (let j = 0; j < 5; j++) {
                if (i !== currentClass && j !== currentClass) {
                    tn += confusionMatrix[i][j];
                }
            }
        }

        // Создаем ячейки бинарной матрицы
        const tpCell = document.createElement('div');
        tpCell.className = 'cell binary-cell tp';
        tpCell.style.gridColumn = '4';
        tpCell.style.gridRow = '3';
        tpCell.textContent = `TP = ${tp}`;
        tpCell.dataset.type = 'tp';
        container.appendChild(tpCell);

        const fnCell = document.createElement('div');
        fnCell.className = 'cell binary-cell fn';
        fnCell.style.gridColumn = '5';
        fnCell.style.gridRow = '3';
        fnCell.textContent = `FN = ${fn}`;
        fnCell.dataset.type = 'fn';
        container.appendChild(fnCell);

        const fpCell = document.createElement('div');
        fpCell.className = 'cell binary-cell fp';
        fpCell.style.gridColumn = '4';
        fpCell.style.gridRow = '4';
        fpCell.textContent = `FP = ${fp}`;
        fpCell.dataset.type = 'fp';
        container.appendChild(fpCell);

        const tnCell = document.createElement('div');
        tnCell.className = 'cell binary-cell tn';
        tnCell.style.gridColumn = '5';
        tnCell.style.gridRow = '4';
        tnCell.textContent = `TN = ${tn}`;
        tnCell.dataset.type = 'tn';
        container.appendChild(tnCell);

        // Добавляем обработчики событий
        addBinaryMatrixEvents();
    }

    function addBinaryMatrixEvents() {
        const binaryCells = document.querySelectorAll('.binary-cell');
        
        binaryCells.forEach(cell => {
            cell.addEventListener('mouseenter', () => highlightMulticlass(cell.dataset.type));
            cell.addEventListener('mouseleave', () => clearHighlight());
        });
    }

    function highlightMulticlass(type) {
        const multiclassCells = document.querySelectorAll('#multiclassMatrix@0 .data-cell');
        
        multiclassCells.forEach(cell => {
            const row = parseInt(cell.dataset.row);
            const col = parseInt(cell.dataset.col);
            
            cell.classList.remove('highlight-tp', 'highlight-fp', 'highlight-fn', 'highlight-tn');
            
            if (type === 'tp' && row === currentClass && col === currentClass) {
                cell.classList.add('highlight-tp');
            } else if (type === 'fp' && row !== currentClass && col === currentClass) {
                cell.classList.add('highlight-fp');
            } else if (type === 'fn' && row === currentClass && col !== currentClass) {
                cell.classList.add('highlight-fn');
            } else if (type === 'tn' && row !== currentClass && col !== currentClass) {
                cell.classList.add('highlight-tn');
            }
        });
    }

    function clearHighlight() {
        const multiclassCells = document.querySelectorAll('#multiclassMatrix@0 .data-cell');
        multiclassCells.forEach(cell => {
            cell.classList.remove('highlight-tp', 'highlight-fp', 'highlight-fn', 'highlight-tn');
        });
    }

    function updateMatrices() {
        createMulticlassMatrix();
        createBinaryMatrix();
    }

    // Обработчик изменения класса
    document.getElementById('classSelect@0').addEventListener('change', (e) => {
        currentClass = parseInt(e.target.value);
        updateMatrices();
    });

    // Инициализация
    createMulticlassMatrix();
    createBinaryMatrix();
</script>

@end
-->

# conf_matrix

@conf_matrix(0)

```
@conf_matrix(
    0               // ID элемента
)
```
