<!--
link: ./bayes.css

@bayes
<div id="bayes-simulator-container"></div>

<script>
'use strict';

// Создаем изолированное пространство имен
window.BayesSimulator = (function() {
  
  // Состояние симулятора
  const state = {
    covarianceType: 'shared_scalar',
    mu1: [180, 200],
    mu2: [420, 200],
    prior1: 0.5,
    prior2: 0.5,
    sharedScalar: 40,
    sharedDiagonal: [50, 30],
    sharedAngle: 0,
    sharedFull: [[2500, 800], [800, 1600]],
    sigma1Scalar: 45,
    sigma2Scalar: 35,
    sigma1Diag: [50, 30],
    sigma2Diag: [40, 35],
    angle1: 15,
    angle2: -20,
    sigma1Full: [[2500, 500], [500, 900]],
    sigma2Full: [[2200, -600], [-600, 1100]],
    dataPoints: [],
    numPoints: 100,
    isDragging: null,
    canvas: null,
    ctx: null
  };

  // Математические функции
  function gaussianRandom() {
    let u = 0, v = 0;
    while(u === 0) u = Math.random();
    while(v === 0) v = Math.random();
    return Math.sqrt(-2.0 * Math.log(u)) * Math.cos(2 * Math.PI * v);
  }

  function matrixMultiply(A, B) {
    const m = A.length, n = A[0].length, p = B[0].length;
    const C = Array(m).fill().map(() => Array(p).fill(0));
    for(let i = 0; i < m; i++) {
      for(let j = 0; j < p; j++) {
        for(let k = 0; k < n; k++) {
          C[i][j] += A[i][k] * B[k][j];
        }
      }
    }
    return C;
  }

  function matrixTranspose(A) {
    return A[0].map((_, i) => A.map(row => row[i]));
  }

  function matrixInverse2x2(A) {
    const det = A[0][0] * A[1][1] - A[0][1] * A[1][0];
    if(Math.abs(det) < 1e-10) return null;
    return [
      [A[1][1]/det, -A[0][1]/det],
      [-A[1][0]/det, A[0][0]/det]
    ];
  }

  function matrixDeterminant2x2(A) {
    return A[0][0] * A[1][1] - A[0][1] * A[1][0];
  }

  function choleskyDecomposition(matrix) {
    const a = matrix[0][0];
    const b = matrix[0][1];
    const c = matrix[1][1];
    
    const l11 = Math.sqrt(a);
    const l21 = b / l11;
    const l22 = Math.sqrt(c - l21 * l21);
    
    return [[l11, 0], [l21, l22]];
  }

  function vectorAdd(a, b) {
    return [a[0] + b[0], a[1] + b[1]];
  }

  function vectorSubtract(a, b) {
    return [a[0] - b[0], a[1] - b[1]];
  }

  function matrixVectorMultiply(M, v) {
    return [
      M[0][0] * v[0] + M[0][1] * v[1],
      M[1][0] * v[0] + M[1][1] * v[1]
    ];
  }

  // Создание ковариационной матрицы
  function getCovarianceMatrix(classIndex) {
    let baseSigma, angle = 0;
    
    const rotateMatrix = (sigma, angleDeg) => {
      if (angleDeg === 0) return sigma;
      const angleRad = (angleDeg * Math.PI) / 180;
      const cos = Math.cos(angleRad);
      const sin = Math.sin(angleRad);
      const R = [[cos, -sin], [sin, cos]];
      const RT = matrixTranspose(R);
      return matrixMultiply(matrixMultiply(R, sigma), RT);
    };
    
    switch (state.covarianceType) {
      case 'shared_scalar':
        return [[state.sharedScalar**2, 0], [0, state.sharedScalar**2]];
        
      case 'shared_diagonal':
        baseSigma = [[state.sharedDiagonal[0]**2, 0], [0, state.sharedDiagonal[1]**2]];
        return rotateMatrix(baseSigma, state.sharedAngle);
        
      case 'shared':
        return state.sharedFull;
        
      case 'scalar':
        const sigma = classIndex === 1 ? state.sigma1Scalar : state.sigma2Scalar;
        return [[sigma**2, 0], [0, sigma**2]];
        
      case 'diagonal':
        const sigmaDiag = classIndex === 1 ? state.sigma1Diag : state.sigma2Diag;
        angle = classIndex === 1 ? state.angle1 : state.angle2;
        baseSigma = [[sigmaDiag[0]**2, 0], [0, sigmaDiag[1]**2]];
        return rotateMatrix(baseSigma, angle);
        
      case 'full':
        return classIndex === 1 ? state.sigma1Full : state.sigma2Full;
        
      default:
        return [[2500, 0], [0, 2500]];
    }
  }

  function isSharedCovariance() {
    const type = state.covarianceType;
    return type === 'shared_scalar' || type === 'shared_diagonal' || type === 'shared';
  }

  // Генерация точек
  function generateMultivariateNormal(mean, covariance) {
    const z1 = gaussianRandom();
    const z2 = gaussianRandom();
    const L = choleskyDecomposition(covariance);
    const transformed = matrixVectorMultiply(L, [z1, z2]);
    return vectorAdd(mean, transformed);
  }

  function generateRandomPoints() {
    const points = [];
    const n1 = Math.floor(state.numPoints * state.prior1);
    const n2 = state.numPoints - n1;
    
    for (let i = 0; i < n1; i++) {
      const point = generateMultivariateNormal(state.mu1, getCovarianceMatrix(1));
      points.push({ x: point[0], y: point[1], class: 1 });
    }
    
    for (let i = 0; i < n2; i++) {
      const point = generateMultivariateNormal(state.mu2, getCovarianceMatrix(2));
      points.push({ x: point[0], y: point[1], class: 2 });
    }
    
    state.dataPoints = points;
  }

  // Вычисление плотности
  function multivariateNormalPDF(x, mean, covariance) {
    const diff = vectorSubtract(x, mean);
    const inv_cov = matrixInverse2x2(covariance);
    const det_cov = matrixDeterminant2x2(covariance);
    
    if (!inv_cov || det_cov <= 0) return 0;
    
    const mahalanobis = diff[0] * (inv_cov[0][0] * diff[0] + inv_cov[0][1] * diff[1]) +
                        diff[1] * (inv_cov[1][0] * diff[0] + inv_cov[1][1] * diff[1]);
    const coefficient = 1 / Math.sqrt(4 * Math.PI * Math.PI * det_cov);
    
    return coefficient * Math.exp(-0.5 * mahalanobis);
  }

  function posteriorProbability(x, mu, sigma, prior) {
    const likelihood = multivariateNormalPDF(x, mu, sigma);
    return likelihood * prior;
  }

  // Визуализация
  function drawVisualization() {
    const ctx = state.ctx;
    const canvas = state.canvas;
    if (!ctx || !canvas) return;
    
    const width = canvas.width;
    const height = canvas.height;
    
    // Очистка
    ctx.fillStyle = '#fafafa';
    ctx.fillRect(0, 0, width, height);
    
    // Отрисовка областей решений
    drawDecisionRegions(ctx, width, height);
    
    // Отрисовка контуров
    drawDensityContours(ctx, width, height);
    
    // Отрисовка границы
    drawDecisionBoundary(ctx, width, height);
    
    // Отрисовка точек
    drawDataPoints(ctx);
    
    // Отрисовка центров
    drawClassCenters(ctx);
  }

  function drawDecisionRegions(ctx, width, height) {
    const step = 4;
    const sigma1 = getCovarianceMatrix(1);
    const sigma2 = getCovarianceMatrix(2);
    
    for (let x = 0; x < width; x += step) {
      for (let y = 0; y < height; y += step) {
        const post1 = posteriorProbability([x, y], state.mu1, sigma1, state.prior1);
        const post2 = posteriorProbability([x, y], state.mu2, sigma2, state.prior2);
        
        ctx.fillStyle = post1 > post2 ? 'rgba(255,200,200,0.15)' : 'rgba(200,200,255,0.15)';
        ctx.fillRect(x, y, step, step);
      }
    }
  }

  function drawDensityContours(ctx, width, height) {
    const sigma1 = getCovarianceMatrix(1);
    const sigma2 = getCovarianceMatrix(2);
    
    ctx.strokeStyle = '#cc0000';
    ctx.lineWidth = 1.5;
    drawEllipseContours(ctx, state.mu1, sigma1, [1.0, 2.0]);
    
    ctx.strokeStyle = '#0000cc';
    ctx.lineWidth = 1.5;
    drawEllipseContours(ctx, state.mu2, sigma2, [1.0, 2.0]);
  }

  function drawEllipseContours(ctx, center, covariance, levels) {
    levels.forEach((level, index) => {
      try {
        const [[a, b], [c, d]] = covariance;
        const trace = a + d;
        const det = a * d - b * c;
        
        if (det <= 0) return;
        
        const discriminant = Math.sqrt(trace * trace - 4 * det);
        const eigenvalue1 = (trace + discriminant) / 2;
        const eigenvalue2 = (trace - discriminant) / 2;
        
        if (eigenvalue1 <= 0 || eigenvalue2 <= 0) return;
        
        const angle = b === 0 ? 0 : Math.atan2(eigenvalue1 - a, b);
        const radiusX = Math.sqrt(eigenvalue1 * level * 2);
        const radiusY = Math.sqrt(eigenvalue2 * level * 2);
        
        ctx.save();
        ctx.globalAlpha = 0.6 - index * 0.15;
        ctx.translate(center[0], center[1]);
        ctx.rotate(angle);
        
        ctx.beginPath();
        ctx.ellipse(0, 0, radiusX, radiusY, 0, 0, 2 * Math.PI);
        ctx.stroke();
        
        ctx.restore();
      } catch (e) {
        console.error('Error drawing ellipse:', e);
      }
    });
  }

  function drawDecisionBoundary(ctx, width, height) {
    const sigma1 = getCovarianceMatrix(1);
    const sigma2 = getCovarianceMatrix(2);
    
    ctx.strokeStyle = '#333333';
    ctx.lineWidth = 2.5;
    
    if (isSharedCovariance()) {
      // Линейная граница
      try {
        const sigmaSolver = matrixInverse2x2(sigma1);
        if (!sigmaSolver) return;
        
        const diff = vectorSubtract(state.mu1, state.mu2);
        const w = matrixVectorMultiply(sigmaSolver, diff);
        const midpoint = [(state.mu1[0] + state.mu2[0])/2, (state.mu1[1] + state.mu2[1])/2];
        const b = -(w[0] * midpoint[0] + w[1] * midpoint[1]) + Math.log(state.prior1/state.prior2);
        
        if (Math.abs(w[1]) > 1e-10) {
          const x1 = 0;
          const y1 = -(w[0] * x1 + b) / w[1];
          const x2 = width;
          const y2 = -(w[0] * x2 + b) / w[1];
          
          ctx.beginPath();
          ctx.moveTo(x1, y1);
          ctx.lineTo(x2, y2);
          ctx.stroke();
        }
      } catch (e) {
        console.error('Error drawing linear boundary:', e);
      }
    } else {
      // Квадратичная граница
      const points = [];
      for (let x = 0; x < width; x += 3) {
        for (let y = 0; y < height; y += 3) {
          const post1 = posteriorProbability([x, y], state.mu1, sigma1, state.prior1);
          const post2 = posteriorProbability([x, y], state.mu2, sigma2, state.prior2);
          
          const diff = Math.abs(post1 - post2);
          const threshold = Math.max(post1, post2) * 0.05;
          if (diff < threshold && threshold > 1e-10) {
            points.push([x, y]);
          }
        }
      }
      
      ctx.fillStyle = '#555555';
      points.forEach(([x, y]) => {
        ctx.beginPath();
        ctx.arc(x, y, 1.5, 0, 2 * Math.PI);
        ctx.fill();
      });
    }
  }

  function drawDataPoints(ctx) {
    state.dataPoints.forEach(point => {
      ctx.fillStyle = point.class === 1 ? '#cc0000' : '#0000cc';
      ctx.beginPath();
      ctx.arc(point.x, point.y, 2.5, 0, 2 * Math.PI);
      ctx.fill();
      
      ctx.strokeStyle = '#ffffff';
      ctx.lineWidth = 0.8;
      ctx.stroke();
    });
  }

  function drawClassCenters(ctx) {
    // Центр класса 1
    ctx.fillStyle = state.isDragging === 'mu1' ? '#ff0000' : '#cc0000';
    ctx.beginPath();
    ctx.arc(state.mu1[0], state.mu1[1], state.isDragging === 'mu1' ? 10 : 8, 0, 2 * Math.PI);
    ctx.fill();
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 2;
    ctx.stroke();
    
    ctx.fillStyle = '#333333';
    ctx.font = 'bold 12px Arial';
    ctx.fillText('μ₁', state.mu1[0] + 12, state.mu1[1] - 12);
    
    // Центр класса 2
    ctx.fillStyle = state.isDragging === 'mu2' ? '#0000ff' : '#0000cc';
    ctx.beginPath();
    ctx.arc(state.mu2[0], state.mu2[1], state.isDragging === 'mu2' ? 10 : 8, 0, 2 * Math.PI);
    ctx.fill();
    ctx.strokeStyle = '#ffffff';
    ctx.lineWidth = 2;
    ctx.stroke();
    
    ctx.fillStyle = '#333333';
    ctx.font = 'bold 12px Arial';
    ctx.fillText('μ₂', state.mu2[0] + 12, state.mu2[1] - 12);
  }

  // Обработчики мыши
  function handleMouseDown(e) {
    const rect = state.canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    if (Math.sqrt((x - state.mu1[0])**2 + (y - state.mu1[1])**2) < 15) {
      state.isDragging = 'mu1';
      state.canvas.style.cursor = 'grabbing';
    } else if (Math.sqrt((x - state.mu2[0])**2 + (y - state.mu2[1])**2) < 15) {
      state.isDragging = 'mu2';
      state.canvas.style.cursor = 'grabbing';
    }
  }

  function handleMouseMove(e) {
    const rect = state.canvas.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;
    
    if (!state.isDragging) {
      if (Math.sqrt((x - state.mu1[0])**2 + (y - state.mu1[1])**2) < 15 || 
          Math.sqrt((x - state.mu2[0])**2 + (y - state.mu2[1])**2) < 15) {
        state.canvas.style.cursor = 'grab';
      } else {
        state.canvas.style.cursor = 'default';
      }
      return;
    }
    
    const clampedX = Math.max(20, Math.min(state.canvas.width - 20, x));
    const clampedY = Math.max(20, Math.min(state.canvas.height - 20, y));
    
    if (state.isDragging === 'mu1') {
      // Вычисляем смещение центра
      const deltaX = clampedX - state.mu1[0];
      const deltaY = clampedY - state.mu1[1];
      
      // Обновляем центр
      state.mu1 = [clampedX, clampedY];
      
      // Перемещаем точки класса 1
      state.dataPoints.forEach(point => {
        if (point.class === 1) {
          point.x += deltaX;
          point.y += deltaY;
        }
      });
    } else if (state.isDragging === 'mu2') {
      // Вычисляем смещение центра
      const deltaX = clampedX - state.mu2[0];
      const deltaY = clampedY - state.mu2[1];
      
      // Обновляем центр
      state.mu2 = [clampedX, clampedY];
      
      // Перемещаем точки класса 2
      state.dataPoints.forEach(point => {
        if (point.class === 2) {
          point.x += deltaX;
          point.y += deltaY;
        }
      });
    }
    
    drawVisualization();
  }

  function handleMouseUp() {
    state.isDragging = null;
    if (state.canvas) {
      state.canvas.style.cursor = 'default';
    }
    updateInfo();
  }

  function getCorrelation(sigma) {
    const [[a, b], [c, d]] = sigma;
    return b / Math.sqrt(a * d);
  }

  function computeAccuracy() {
    let correct = 0;
    const sigma1 = getCovarianceMatrix(1);
    const sigma2 = getCovarianceMatrix(2);
    
    state.dataPoints.forEach(point => {
      const post1 = posteriorProbability([point.x, point.y], state.mu1, sigma1, state.prior1);
      const post2 = posteriorProbability([point.x, point.y], state.mu2, sigma2, state.prior2);
      const predicted = post1 > post2 ? 1 : 2;
      if (predicted === point.class) correct++;
    });
    
    return state.dataPoints.length > 0 ? (correct / state.dataPoints.length * 100).toFixed(1) : '0';
  }

  function updateInfo() {
    const boundaryType = isSharedCovariance() ? 'Линейная' : 'Квадратичная';
    const accuracy = computeAccuracy();
    const distance = Math.sqrt(
      (state.mu1[0] - state.mu2[0])**2 + 
      (state.mu1[1] - state.mu2[1])**2
    ).toFixed(1);
    
    const boundaryEl = document.getElementById('boundary-type');
    const accuracyEl = document.getElementById('accuracy');
    const distanceEl = document.getElementById('distance');
    
    if (boundaryEl) boundaryEl.textContent = boundaryType;
    if (accuracyEl) accuracyEl.textContent = accuracy;
    if (distanceEl) distanceEl.textContent = distance;
  }

  function updateCovarianceControls() {
    const container = document.getElementById('covariance-controls');
    if (!container) return;
    
    const type = state.covarianceType;
    let html = '';
    
    if (type === 'shared_scalar') {
      html = `
        <label class="bayes-slider-label">
          Общая дисперсия σ: <span id="shared-scalar-value">${state.sharedScalar}</span>
          <input type="range" id="shared-scalar" min="20" max="80" value="${state.sharedScalar}" class="bayes-slider">
        </label>
      `;
    } else if (type === 'shared_diagonal') {
      html = `
        <label class="bayes-slider-label">
          σ₁ (горизонталь): <span id="shared-diag1-value">${state.sharedDiagonal[0]}</span>
          <input type="range" id="shared-diag1" min="20" max="100" value="${state.sharedDiagonal[0]}" class="bayes-slider">
        </label>
        <label class="bayes-slider-label">
          σ₂ (вертикаль): <span id="shared-diag2-value">${state.sharedDiagonal[1]}</span>
          <input type="range" id="shared-diag2" min="20" max="100" value="${state.sharedDiagonal[1]}" class="bayes-slider">
        </label>
        <label class="bayes-slider-label">
          🔄 Угол поворота: <span id="shared-angle-value">${state.sharedAngle}°</span>
          <input type="range" id="shared-angle" min="-90" max="90" value="${state.sharedAngle}" class="bayes-slider">
        </label>
      `;
    } else if (type === 'shared') {
      const corr = getCorrelation(state.sharedFull);
      html = `
        <label class="bayes-slider-label">
          Дисперсия σ₁₁: <span id="shared-var1-value">${Math.sqrt(state.sharedFull[0][0]).toFixed(0)}</span>
          <input type="range" id="shared-var1" min="1000" max="5000" value="${state.sharedFull[0][0]}" class="bayes-slider">
        </label>
        <label class="bayes-slider-label">
          Дисперсия σ₂₂: <span id="shared-var2-value">${Math.sqrt(state.sharedFull[1][1]).toFixed(0)}</span>
          <input type="range" id="shared-var2" min="500" max="3000" value="${state.sharedFull[1][1]}" class="bayes-slider">
        </label>
        <label class="bayes-slider-label">
          📈 Корреляция: <span id="shared-corr-value">${corr.toFixed(2)}</span>
          <input type="range" id="shared-corr" min="-1500" max="1500" value="${state.sharedFull[0][1]}" class="bayes-slider">
        </label>
      `;
    } else if (type === 'scalar') {
      html = `
        <label class="bayes-slider-label">
          σ₁ (класс 1): <span id="sigma1-scalar-value">${state.sigma1Scalar}</span>
          <input type="range" id="sigma1-scalar" min="20" max="100" value="${state.sigma1Scalar}" class="bayes-slider">
        </label>
        <label class="bayes-slider-label">
          σ₂ (класс 2): <span id="sigma2-scalar-value">${state.sigma2Scalar}</span>
          <input type="range" id="sigma2-scalar" min="20" max="100" value="${state.sigma2Scalar}" class="bayes-slider">
        </label>
      `;
    } else if (type === 'diagonal') {
      html = `
        <div class="bayes-class-section">
          <div class="bayes-class-label">Класс 1 (красный):</div>
          <label class="bayes-slider-label">
            σ₁₁: <span id="sigma1-diag1-value">${state.sigma1Diag[0]}</span>
            <input type="range" id="sigma1-diag1" min="20" max="100" value="${state.sigma1Diag[0]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            σ₁₂: <span id="sigma1-diag2-value">${state.sigma1Diag[1]}</span>
            <input type="range" id="sigma1-diag2" min="20" max="100" value="${state.sigma1Diag[1]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            🔄 Угол: <span id="angle1-value">${state.angle1}°</span>
            <input type="range" id="angle1" min="-90" max="90" value="${state.angle1}" class="bayes-slider">
          </label>
        </div>
        
        <div class="bayes-class-section">
          <div class="bayes-class-label">Класс 2 (синий):</div>
          <label class="bayes-slider-label">
            σ₂₁: <span id="sigma2-diag1-value">${state.sigma2Diag[0]}</span>
            <input type="range" id="sigma2-diag1" min="20" max="100" value="${state.sigma2Diag[0]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            σ₂₂: <span id="sigma2-diag2-value">${state.sigma2Diag[1]}</span>
            <input type="range" id="sigma2-diag2" min="20" max="100" value="${state.sigma2Diag[1]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            🔄 Угол: <span id="angle2-value">${state.angle2}°</span>
            <input type="range" id="angle2" min="-90" max="90" value="${state.angle2}" class="bayes-slider">
          </label>
        </div>
      `;
    } else if (type === 'full') {
      const corr1 = getCorrelation(state.sigma1Full);
      const corr2 = getCorrelation(state.sigma2Full);
      html = `
        <div class="bayes-class-section">
          <div class="bayes-class-label">Класс 1 (красный):</div>
          <label class="bayes-slider-label">
            Дисперсия σ₁₁: <span id="s1-var1-value">${Math.sqrt(state.sigma1Full[0][0]).toFixed(0)}</span>
            <input type="range" id="s1-var1" min="1000" max="5000" value="${state.sigma1Full[0][0]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            Дисперсия σ₂₂: <span id="s1-var2-value">${Math.sqrt(state.sigma1Full[1][1]).toFixed(0)}</span>
            <input type="range" id="s1-var2" min="500" max="3000" value="${state.sigma1Full[1][1]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            📈 Корреляция: <span id="s1-corr-value">${corr1.toFixed(2)}</span>
            <input type="range" id="s1-corr" min="-1500" max="1500" value="${state.sigma1Full[0][1]}" class="bayes-slider">
          </label>
        </div>
        
        <div class="bayes-class-section">
          <div class="bayes-class-label">Класс 2 (синий):</div>
          <label class="bayes-slider-label">
            Дисперсия σ₁₁: <span id="s2-var1-value">${Math.sqrt(state.sigma2Full[0][0]).toFixed(0)}</span>
            <input type="range" id="s2-var1" min="1000" max="5000" value="${state.sigma2Full[0][0]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            Дисперсия σ₂₂: <span id="s2-var2-value">${Math.sqrt(state.sigma2Full[1][1]).toFixed(0)}</span>
            <input type="range" id="s2-var2" min="500" max="3000" value="${state.sigma2Full[1][1]}" class="bayes-slider">
          </label>
          <label class="bayes-slider-label">
            📈 Корреляция: <span id="s2-corr-value">${corr2.toFixed(2)}</span>
            <input type="range" id="s2-corr" min="-1500" max="1500" value="${state.sigma2Full[0][1]}" class="bayes-slider">
          </label>
        </div>
      `;
    }
    
    container.innerHTML = html;
    attachCovarianceListeners();
  }

  function attachCovarianceListeners() {
    const type = state.covarianceType;
    
    if (type === 'shared_scalar') {
      const slider = document.getElementById('shared-scalar');
      if (slider) {
        slider.addEventListener('input', (e) => {
          state.sharedScalar = parseInt(e.target.value);
          document.getElementById('shared-scalar-value').textContent = state.sharedScalar;
          drawVisualization();
          updateInfo();
        });
      }
    } else if (type === 'shared_diagonal') {
      ['shared-diag1', 'shared-diag2', 'shared-angle'].forEach((id, idx) => {
        const slider = document.getElementById(id);
        if (slider) {
          slider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            if (idx === 0) state.sharedDiagonal[0] = val;
            else if (idx === 1) state.sharedDiagonal[1] = val;
            else state.sharedAngle = val;
            
            document.getElementById(id + '-value').textContent = 
              idx === 2 ? val + '°' : val;
            drawVisualization();
            updateInfo();
          });
        }
      });
    } else if (type === 'shared') {
      ['shared-var1', 'shared-var2', 'shared-corr'].forEach((id, idx) => {
        const slider = document.getElementById(id);
        if (slider) {
          slider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            if (idx === 0) state.sharedFull[0][0] = val;
            else if (idx === 1) state.sharedFull[1][1] = val;
            else {
              state.sharedFull[0][1] = val;
              state.sharedFull[1][0] = val;
            }
            
            if (idx < 2) {
              document.getElementById(id + '-value').textContent = Math.sqrt(val).toFixed(0);
            } else {
              const corr = getCorrelation(state.sharedFull);
              document.getElementById(id + '-value').textContent = corr.toFixed(2);
            }
            drawVisualization();
            updateInfo();
          });
        }
      });
    } else if (type === 'scalar') {
      ['sigma1-scalar', 'sigma2-scalar'].forEach((id, idx) => {
        const slider = document.getElementById(id);
        if (slider) {
          slider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            if (idx === 0) state.sigma1Scalar = val;
            else state.sigma2Scalar = val;
            
            document.getElementById(id + '-value').textContent = val;
            drawVisualization();
            updateInfo();
          });
        }
      });
    } else if (type === 'diagonal') {
      const controls = [
        ['sigma1-diag1', 0, 0], ['sigma1-diag2', 0, 1], ['angle1', 0, 2],
        ['sigma2-diag1', 1, 0], ['sigma2-diag2', 1, 1], ['angle2', 1, 2]
      ];
      
      controls.forEach(([id, classIdx, paramIdx]) => {
        const slider = document.getElementById(id);
        if (slider) {
          slider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            if (classIdx === 0) {
              if (paramIdx === 0) state.sigma1Diag[0] = val;
              else if (paramIdx === 1) state.sigma1Diag[1] = val;
              else state.angle1 = val;
            } else {
              if (paramIdx === 0) state.sigma2Diag[0] = val;
              else if (paramIdx === 1) state.sigma2Diag[1] = val;
              else state.angle2 = val;
            }
            
            document.getElementById(id + '-value').textContent = 
              paramIdx === 2 ? val + '°' : val;
            drawVisualization();
            updateInfo();
          });
        }
      });
    } else if (type === 'full') {
      const controls = [
        ['s1-var1', 0, 0], ['s1-var2', 0, 1], ['s1-corr', 0, 2],
        ['s2-var1', 1, 0], ['s2-var2', 1, 1], ['s2-corr', 1, 2]
      ];
      
      controls.forEach(([id, classIdx, paramIdx]) => {
        const slider = document.getElementById(id);
        if (slider) {
          slider.addEventListener('input', (e) => {
            const val = parseInt(e.target.value);
            const sigma = classIdx === 0 ? state.sigma1Full : state.sigma2Full;
            
            if (paramIdx === 0) sigma[0][0] = val;
            else if (paramIdx === 1) sigma[1][1] = val;
            else {
              sigma[0][1] = val;
              sigma[1][0] = val;
            }
            
            if (paramIdx < 2) {
              document.getElementById(id + '-value').textContent = Math.sqrt(val).toFixed(0);
            } else {
              const corr = getCorrelation(sigma);
              document.getElementById(id + '-value').textContent = corr.toFixed(2);
            }
            
            drawVisualization();
            updateInfo();
          });
        }
      });
    }
  }

  function attachEventListeners() {
    // Canvas события
    state.canvas.addEventListener('mousedown', handleMouseDown);
    state.canvas.addEventListener('mousemove', handleMouseMove);
    state.canvas.addEventListener('mouseup', handleMouseUp);
    state.canvas.addEventListener('mouseleave', handleMouseUp);
    
    // Тип ковариации
    document.querySelectorAll('input[name="covariance"]').forEach(radio => {
      radio.addEventListener('change', (e) => {
        state.covarianceType = e.target.value;
        updateCovarianceControls();
        generateRandomPoints();
        drawVisualization();
        updateInfo();
      });
    });
    
    // Априорные вероятности
    const prior1Slider = document.getElementById('prior1-slider');
    if (prior1Slider) {
      prior1Slider.addEventListener('input', (e) => {
        const p1 = parseFloat(e.target.value);
        state.prior1 = p1;
        state.prior2 = 1 - p1;
        document.getElementById('prior1-value').textContent = p1.toFixed(2);
        document.getElementById('prior2-value').textContent = state.prior2.toFixed(2);
        generateRandomPoints();
        drawVisualization();
        updateInfo();
      });
    }
    
    // Количество точек
    const numpointsSlider = document.getElementById('numpoints-slider');
    if (numpointsSlider) {
      numpointsSlider.addEventListener('input', (e) => {
        state.numPoints = parseInt(e.target.value);
        document.getElementById('numpoints-value').textContent = state.numPoints;
        generateRandomPoints();
        drawVisualization();
        updateInfo();
      });
    }
    
    // Кнопка генерации
    const generateBtn = document.getElementById('generate-btn');
    if (generateBtn) {
      generateBtn.addEventListener('click', () => {
        generateRandomPoints();
        drawVisualization();
        updateInfo();
      });
    }
  }

  function createUI() {
    const container = document.getElementById('bayes-simulator-container');
    if (!container) return;
    
    container.innerHTML = `
      <div class="bayes-simulator">
        <div class="bayes-header">
          <h3>Байесовская классификация: типы границ решений</h3>
          <p class="bayes-subtitle">Перетаскивайте центры классов • Изменяйте параметры ковариации</p>
        </div>
        
        <div class="bayes-content">
          <div class="bayes-covariance-selector">
            <label class="bayes-label">Структура ковариационной матрицы:</label>
            <div class="bayes-radio-group">
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="shared_scalar" checked>
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Shared Scalar</div>
                  <div class="bayes-radio-formula">Σ₁ = Σ₂ = σ²I</div>
                </div>
              </label>
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="shared_diagonal">
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Shared Diagonal</div>
                  <div class="bayes-radio-formula">Σ₁ = Σ₂ = diag(σ₁², σ₂²)</div>
                </div>
              </label>
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="shared">
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Shared</div>
                  <div class="bayes-radio-formula">Σ₁ = Σ₂ = Σ</div>
                </div>
              </label>
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="scalar">
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Scalar</div>
                  <div class="bayes-radio-formula">Σₖ = σₖ²I</div>
                </div>
              </label>
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="diagonal">
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Diagonal</div>
                  <div class="bayes-radio-formula">Σₖ = diag(σₖ₁², σₖ₂²)</div>
                </div>
              </label>
              <label class="bayes-radio-item">
                <input type="radio" name="covariance" value="full">
                <div class="bayes-radio-label">
                  <div class="bayes-radio-title">Full</div>
                  <div class="bayes-radio-formula">Σₖ - произвольные</div>
                </div>
              </label>
            </div>
          </div>
          
          <canvas id="bayes-canvas" width="600" height="400" class="bayes-canvas"></canvas>
          
          <div class="bayes-controls">
            <div class="bayes-control-group">
              <h4 class="bayes-control-title">Параметры ковариации</h4>
              <div id="covariance-controls"></div>
            </div>
            
            <div class="bayes-control-group">
              <h4 class="bayes-control-title">Параметры модели</h4>
              
              <label class="bayes-slider-label">
                Априорная вероятность P(Y=1): <span id="prior1-value">0.50</span>
                <input type="range" id="prior1-slider" min="0.1" max="0.9" step="0.05" value="0.5" class="bayes-slider">
                <div class="bayes-slider-info">P(Y=2) = <span id="prior2-value">0.50</span></div>
              </label>
              
              <label class="bayes-slider-label">
                Количество точек: <span id="numpoints-value">100</span>
                <input type="range" id="numpoints-slider" min="50" max="200" value="100" class="bayes-slider">
              </label>
              
              <button id="generate-btn" class="bayes-button">🎲 Сгенерировать новые данные</button>
              
              <div class="bayes-info-box">
                <div><strong>Тип границы:</strong> <span id="boundary-type">Линейная</span></div>
                <div><strong>Точность:</strong> <span id="accuracy">0</span>%</div>
                <div><strong>Расстояние между центрами:</strong> <span id="distance">0</span></div>
              </div>
            </div>
          </div>
        </div>
      </div>
    `;
    
    // Инициализация canvas
    state.canvas = document.getElementById('bayes-canvas');
    state.ctx = state.canvas.getContext('2d');
    
    // Добавление обработчиков
    attachEventListeners();
    
    // Обновление контролов ковариации
    updateCovarianceControls();
    
    // Генерация данных и отрисовка
    generateRandomPoints();
    drawVisualization();
    updateInfo();
  }

  function init() {
    // Ждем загрузки DOM
    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', createUI);
    } else {
      createUI();
    }
  }

  // Публичный API
  return {
    init: init
  };
})();

// Автоматический запуск
window.BayesSimulator.init();
</script>
@end
-->

# bayes

kjlk

@bayes()