<!--
link: ./fpr_tpr.css

@prec_rec
<div class="fpr_tpr-container">    
    <div class="slider-container">
        <label for="thresholdSlider_@0">Порог:</label>
        <input type="range" id="thresholdSlider_@0" min="-@1" max="@1" step="@2" value="0">
        <span id="sliderValue_@0">0.0</span>
    </div>
    
    <div class="visualization-container">
        <div class="ellipse-container">
            <h3>Recall</h3>
            <div class="metrics-container" id="fprValue_@0">0.00</div>
            <canvas id="fprCanvas_@0" width="250" height="250"></canvas>
        </div>
        
        <div class="ellipse-container">
            <h3>Precision</h3>
            <div class="metrics-container" id="tprValue_@0">0.00</div>
            <canvas id="tprCanvas_@0" width="250" height="250"></canvas>
        </div>
    </div>
</div>

<script>
    // Configuration parameters
    const config = {
        sliderMin: -1 * parseFloat("@1"),
        sliderMax: parseFloat("@1"),
        sliderStep: parseFloat("@2"),
        angleDeg: @3,
        a: 1.3,        // ellipse width
        bEllipse: 0.8, // ellipse height
        canvasSize: 250,
        plotRange: 1.5,
    };

    // Color configurations
    const fprColors = { //recall
        tpColor: [0, 125, 187],
        fnColor: [255, 170, 79],
        fpColor: [242, 242, 242],
        tnColor: [242, 242, 242]
    };

    const tprColors = { // precision
        tpColor: [0, 125, 187],
        fnColor: [242, 242, 242],
        fpColor: [158, 219, 235],
        tnColor: [242, 242, 242]
    };

    // Get DOM elements
    const fprCanvas = document.getElementById('fprCanvas_@0');
    const tprCanvas = document.getElementById('tprCanvas_@0');
    const fprCtx = fprCanvas.getContext('2d');
    const tprCtx = tprCanvas.getContext('2d');
    const slider = document.getElementById('thresholdSlider_@0');
    const sliderValue = document.getElementById('sliderValue_@0');
    const fprValueElement = document.getElementById('fprValue_@0');
    const tprValueElement = document.getElementById('tprValue_@0');

    // Set slider properties from config
    slider.min = config.sliderMin;
    slider.max = config.sliderMax;
    slider.step = config.sliderStep;
    slider.value = config.add;
    sliderValue.textContent = config.add;

    // Scale factor from data coordinates to pixels
    const scale = config.canvasSize / (2 * config.plotRange);

    // Convert data coordinates to canvas pixels
    function toPixelX(x) {
        return (x + config.plotRange) * scale;
    }

    function toPixelY(y) {
        return config.canvasSize - (y + config.plotRange) * scale;
    }

    // Calculate sector areas and centers
    function calculateSectorInfo(add) {
        const theta = config.angleDeg * Math.PI / 180;
        const slope = Math.tan(theta);
        const centers = { tp: [0, 0], fn: [0, 0], fp: [0, 0], tn: [0, 0] };
        const counts = { tp: 0, fn: 0, fp: 0, tn: 0 };
        const step = 0.01;
        
        // Sample points within ellipse to find sector centers and counts
        for (let x = -config.a; x <= config.a; x += step) {
            for (let y = -config.bEllipse; y <= config.bEllipse; y += step) {
                if ((x*x)/(config.a*config.a) + (y*y)/(config.bEllipse*config.bEllipse) <= 1) {
                    if (y < 0 && y > slope * x + add) {
                        // FP sector
                        centers.fp[0] += x;
                        centers.fp[1] += y;
                        counts.fp++;
                    } else if (y > 0 && y > slope * x + add) {
                        // TP sector
                        centers.tp[0] += x;
                        centers.tp[1] += y;
                        counts.tp++;
                    } else if (y < 0 && y < slope * x + add) {
                        // TN sector
                        centers.tn[0] += x;
                        centers.tn[1] += y;
                        counts.tn++;
                    } else if (y > 0 && y < slope * x + add) {
                        // FN sector
                        centers.fn[0] += x;
                        centers.fn[1] += y;
                        counts.fn++;
                    }
                }
            }
        }
        
        // Calculate average positions
        if (counts.tp > 0) {
            centers.tp[0] /= counts.tp;
            centers.tp[1] /= counts.tp;
        }
        if (counts.fn > 0) {
            centers.fn[0] /= counts.fn;
            centers.fn[1] /= counts.fn;
        }
        if (counts.fp > 0) {
            centers.fp[0] /= counts.fp;
            centers.fp[1] /= counts.fp;
        }
        if (counts.tn > 0) {
            centers.tn[0] /= counts.tn;
            centers.tn[1] /= counts.tn;
        }
        
        // Calculate total points (approximate area)
        const totalPoints = counts.tp + counts.fn + counts.fp + counts.tn;
        
        return {
            centers: centers,
            counts: counts,
            totalPoints: totalPoints
        };
    }

    // Draw the ellipse visualization
    function plotEllipse(ctx, colors, add, isFPR) {
        // Clear canvas
        ctx.clearRect(0, 0, config.canvasSize, config.canvasSize);
        
        // Calculate parameters
        const theta = config.angleDeg * Math.PI / 180;
        const slope = Math.tan(theta);
        
        // Calculate sector info
        const sectorInfo = calculateSectorInfo(add);
        const centers = sectorInfo.centers;
        const counts = sectorInfo.counts;
        
        // Calculate FPR and TPR
        const fp = counts.fp;
        const tn = counts.tn;
        const tp = counts.tp;
        const fn = counts.fn;
        
        let precision = tp / (tp + fp)
        if (config.angleDeg == 89.99){
            precision = 0.5
        }

        const fpr = tp / (tp + fn);     //recall
        const tpr = precision;     //precision
        
        // Update metric displays
        if (isFPR) {
            fprValueElement.textContent = fpr.toFixed(2);
        } else {
            tprValueElement.textContent = tpr.toFixed(2);
        }
        
        // Draw ellipse outline
        ctx.beginPath();
        ctx.ellipse(
            toPixelX(0), 
            toPixelY(0), 
            config.a * scale, 
            config.bEllipse * scale, 
            0, 0, 2 * Math.PI
        );
        ctx.strokeStyle = 'white';
        ctx.lineWidth = 2;
        ctx.stroke();
        
        // Create an off-screen canvas for pixel manipulation
        const pixelCanvas = document.createElement('canvas');
        pixelCanvas.width = config.canvasSize;
        pixelCanvas.height = config.canvasSize;
        const pixelCtx = pixelCanvas.getContext('2d');
        const imageData = pixelCtx.createImageData(config.canvasSize, config.canvasSize);
        const data = imageData.data;
        
        // Sample points and color sectors
        //const step = 2 * config.plotRange / config.canvasSize;
        for (let i = 0; i < config.canvasSize; i++) {
            for (let j = 0; j < config.canvasSize; j++) {
                // Convert pixel to data coordinates
                const x = (i / scale) - config.plotRange;
                const y = config.plotRange - (j / scale);
                
                // Check if point is inside ellipse
                const inEllipse = (x*x)/(config.a*config.a) + (y*y)/(config.bEllipse*config.bEllipse) <= 1;
                
                if (inEllipse) {
                    const idx = (j * config.canvasSize + i) * 4;
                    
                    // Determine sector
                    if (y < 0 && y > slope * x + add) {
                        // FP sector
                        data[idx] = colors.fpColor[0];
                        data[idx+1] = colors.fpColor[1];
                        data[idx+2] = colors.fpColor[2];
                    } else if (y > 0 && y > slope * x + add) {
                        // TP sector
                        data[idx] = colors.tpColor[0];
                        data[idx+1] = colors.tpColor[1];
                        data[idx+2] = colors.tpColor[2];
                    } else if (y < 0 && y < slope * x + add) {
                        // TN sector
                        data[idx] = colors.tnColor[0];
                        data[idx+1] = colors.tnColor[1];
                        data[idx+2] = colors.tnColor[2];
                    } else if (y > 0 && y < slope * x + add) {
                        // FN sector
                        data[idx] = colors.fnColor[0];
                        data[idx+1] = colors.fnColor[1];
                        data[idx+2] = colors.fnColor[2];
                    }
                    data[idx+3] = 255; // Alpha channel
                }
            }
        }
        
        // Put the pixel data back to the off-screen canvas
        pixelCtx.putImageData(imageData, 0, 0);
        
        // Draw the off-screen canvas to the main canvas
        ctx.drawImage(pixelCanvas, 0, 0);
        
        // Draw the dividing line
        ctx.beginPath();
        const x1 = -config.plotRange;
        const y1 = slope * x1 + add;
        const x2 = config.plotRange;
        const y2 = slope * x2 + add;
        ctx.moveTo(toPixelX(x1), toPixelY(y1));
        ctx.lineTo(toPixelX(x2), toPixelY(y2));
        ctx.strokeStyle = 'black';
        ctx.lineWidth = 3;
        ctx.stroke();
        
        // Add labels at calculated centers
        if (centers.tp[0] + centers.tp[1] != 0) addLabel(ctx, "TP", centers.tp, colors.tpColor);
        if (centers.fn[0] + centers.fn[1] != 0) addLabel(ctx, "FN", centers.fn, colors.fnColor);
        if (centers.fp[0] + centers.fp[1] != 0) addLabel(ctx, "FP", centers.fp, colors.fpColor);
        if (centers.tn[0] + centers.tn[1] != 0) addLabel(ctx, "TN", centers.tn, colors.tnColor);
    }
    
    // Add label at specific position
    function addLabel(ctx, text, center, color) {
        ctx.fillStyle = 'white';
        ctx.font = 'bold 20px Arial';
        ctx.textAlign = 'center';
        ctx.textBaseline = 'middle';
        ctx.fillText(text, toPixelX(center[0]), toPixelY(center[1]));
    }
    
    // Event listener for slider
    slider.addEventListener('input', function() {
        const value = parseFloat(this.value);
        sliderValue.textContent = value.toFixed(1);
        plotEllipse(fprCtx, fprColors, value, true);
        plotEllipse(tprCtx, tprColors, value, false);
    });
    
    // Initial plot
    plotEllipse(fprCtx, fprColors, 0, true);
    plotEllipse(tprCtx, tprColors, 0, false);
</script>
@end
-->

# prec_rec

@prec_rec(0, 2.4, 0.1, 60)

@prec_rec(1, 0.8, 0.1, 0)

@prec_rec(2, 7500, 1000, 89.99)

```
@prec_rec(
    0,               // ID элемента
    1.0,             // sliderMax
    0.1,             // sliderStep
    60,              // angleDeg
)
```
