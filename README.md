<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Interactive Definite Integral Calculator</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/mathjs/11.8.0/math.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        :root {
            --primary: #2563eb;
            --primary-light: rgba(37, 99, 235, 0.2);
            --bg: #f8fafc;
            --card: #ffffff;
            --text: #1e293b;
            --border: #e2e8f0;
        }

        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background-color: var(--bg);
            color: var(--text);
            margin: 0;
            padding: 20px;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            box-sizing: border-box;
        }

        .container {
            background: var(--card);
            border-radius: 12px;
            box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
            width: 100%;
            max-width: 950px;
            padding: 24px;
            display: grid;
            grid-template-columns: 1fr;
            gap: 24px;
        }

        @media (min-width: 768px) {
            .container {
                grid-template-columns: 350px 1fr;
            }
        }

        .controls-panel {
            display: flex;
            flex-direction: column;
            gap: 20px;
        }

        h1 {
            font-size: 1.5rem;
            margin: 0 0 4px 0;
            color: #0f172a;
        }

        p.subtitle {
            font-size: 0.875rem;
            color: #64748b;
            margin: 0 0 10px 0;
        }

        .input-group {
            display: flex;
            flex-direction: column;
            gap: 6px;
        }

        label {
            font-size: 0.875rem;
            font-weight: 600;
            color: #475569;
        }

        input[type="text"], input[type="number"] {
            padding: 10px 12px;
            border: 1px solid var(--border);
            border-radius: 6px;
            font-size: 1rem;
            outline: none;
            transition: border-color 0.2s;
        }

        input[type="text"]:focus, input[type="number"]:focus {
            border-color: var(--primary);
            box-shadow: 0 0 0 3px var(--primary-light);
        }

        .range-wrapper {
            display: flex;
            align-items: center;
            gap: 10px;
        }

        input[type="range"] {
            flex-grow: 1;
            accent-color: var(--primary);
        }

        .limits-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 12px;
        }

        .result-box {
            background-color: #f1f5f9;
            border-left: 4px solid var(--primary);
            padding: 16px;
            border-radius: 4px;
            margin-top: auto;
        }

        .result-title {
            font-size: 0.75rem;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            color: #64748b;
            font-weight: 700;
        }

        .result-value {
            font-size: 1.5rem;
            font-weight: 700;
            font-family: monospace;
            color: #0f172a;
            margin-top: 4px;
            word-break: break-all;
        }

        .error-message {
            color: #dc2626;
            font-size: 0.85rem;
            margin-top: 4px;
            display: none;
        }

        .graph-panel {
            position: relative;
            min-height: 300px;
            border: 1px solid var(--border);
            border-radius: 8px;
            padding: 8px;
            background: #ffffff;
        }

        canvas {
            width: 100% !important;
            height: 100% !important;
        }
    </style>
</head>
<body>

<div class="container">
    <div class="controls-panel">
        <div>
            <h1>Integral Calculator</h1>
            <p class="subtitle">Simpler Definite Integration Visualizer</p>
        </div>

        <div class="input-group">
            <label for="expression">Define Function f(x)</label>
            <input type="text" id="expression" value="x^2 - 1" placeholder="e.g., x^2 - 1 or sin(x)">
            <div id="error-text" class="error-message">Invalid mathematical expression.</div>
        </div>

        <div class="limits-grid">
            <div class="input-group">
                <label for="limit-a">Lower Bound (a)</label>
                <input type="number" id="limit-a" value="0" step="0.1">
            </div>
            <div class="input-group">
                <label for="limit-b">Upper Bound (b)</label>
                <input type="number" id="limit-b" value="2" step="0.1">
            </div>
        </div>

        <div class="input-group">
            <label>Adjust Lower Bound (a)</label>
            <div class="range-wrapper">
                <input type="range" id="slider-a" min="-5" max="5" step="0.1" value="0">
            </div>
        </div>

        <div class="input-group">
            <label>Adjust Upper Bound (b)</label>
            <div class="range-wrapper">
                <input type="range" id="slider-b" min="-5" max="5" step="0.1" value="2">
            </div>
        </div>

        <div class="result-box">
            <div class="result-title">Definite Integral Value</div>
            <div id="integral-result" class="result-value">0.6667</div>
        </div>
    </div>

    <div class="graph-panel">
        <canvas id="graphCanvas"></canvas>
    </div>
</div>

<script>
    // Elements DOM References
    const formulaInput = document.getElementById('expression');
    const inputA = document.getElementById('limit-a');
    const inputB = document.getElementById('limit-b');
    const sliderA = document.getElementById('slider-a');
    const sliderB = document.getElementById('slider-b');
    const errorText = document.getElementById('error-text');
    const resultDisplay = document.getElementById('integral-result');
    const ctx = document.getElementById('graphCanvas').getContext('2d');

    let chart;

    // Helper to evaluate f(x) safely via math.js
    function evaluateFunction(expr, xValue) {
        try {
            const compiled = math.compile(expr);
            return compiled.evaluate({ x: xValue });
        } catch (e) {
            return null;
        }
    }

    // Simpson's Rule for clean, hyper-accurate numerical Integration calculation
    function calculateIntegral(expr, a, b) {
        const n = 200; // Even number of subintervals
        const h = (b - a) / n;
        let sum = evaluateFunction(expr, a) + evaluateFunction(expr, b);

        for (let i = 1; i < n; i++) {
            const x = a + i * h;
            const val = evaluateFunction(expr, x);
            if (val === null || isNaN(val)) return NaN;
            sum += (i % 2 === 0 ? 2 : 4) * val;
        }
        return (h / 3) * sum;
    }

    // Build or update the visual canvas chart layout
    function updateCalculator() {
        const exprStr = formulaInput.value;
        const a = parseFloat(inputA.value);
        const b = parseFloat(inputB.value);

        // Quick verification check
        let testEval = evaluateFunction(exprStr, 1);
        if (testEval === null || isNaN(testEval)) {
            errorText.style.display = 'block';
            return;
        } else {
            errorText.style.display = 'none';
        }

        // Calculate Definite Integral value
        const integralValue = calculateIntegral(exprStr, a, b);
        if (!isNaN(integralValue)) {
            // Rounded cleanly to 5 decimal places
            resultDisplay.textContent = Number(integralValue.toFixed(5)).toString();
        } else {
            resultDisplay.textContent = "Error";
        }

        // Generate data ranges to feed chart representation (Dynamic viewport based on bounding constraints)
        const xMin = Math.min(a, b, -4) - 1;
        const xMax = Math.max(a, b, 4) + 1;
        const steps = 150;
        const stepSize = (xMax - xMin) / steps;

        const mainLineData = [];
        const shadedAreaData = [];

        for (let i = 0; i <= steps; i++) {
            const x = xMin + i * stepSize;
            const y = evaluateFunction(exprStr, x);
            
            if (y !== null && !isNaN(y)) {
                mainLineData.push({ x: x, y: y });

                // Construct condition to feed shaded integral area range boundary limits [a, b]
                const lowerBound = Math.min(a, b);
                const upperBound = Math.max(a, b);
                if (x >= lowerBound && x <= upperBound) {
                    shadedAreaData.push({ x: x, y: y });
                }
            }
        }

        // Render update or replace chart entirely
        if (chart) {
            chart.data.datasets[0].data = mainLineData;
            chart.data.datasets[1].data = shadedAreaData;
            chart.options.scales.x.min = xMin;
            chart.options.scales.x.max = xMax;
            chart.update('none'); // Update smoothly without canvas snapping animations
        } else {
            chart = new Chart(ctx, {
                type: 'line',
                data: {
                    datasets: [
                        {
                            label: 'f(x)',
                            data: mainLineData,
                            borderColor: '#2563eb',
                            borderWidth: 2.5,
                            pointRadius: 0,
                            fill: false,
                            order: 1
                        },
                        {
                            label: 'Area Under Curve',
                            data: shadedAreaData,
                            backgroundColor: 'rgba(37, 99, 235, 0.25)',
                            borderColor: 'transparent',
                            pointRadius: 0,
                            fill: 'origin', // Shaded cleanly to the absolute y=0 baseline axis
                            order: 2
                        }
                    ]
                },
                options: {
                    responsive: true,
                    maintainAspectRatio: false,
                    plugins: {
                        legend: { display: false }
                    },
                    scales: {
                        x: {
                            type: 'linear',
                            position: 'bottom',
                            grid: { color: '#e2e8f0' }
                        },
                        y: {
                            type: 'linear',
                            grid: { color: '#e2e8f0' }
                        }
                    }
                }
            });
        }
    }

    // Bi-directional event listeners linking sliders to textual context metrics fields
    function syncSlidersToFields() {
        inputA.value = sliderA.value;
        inputB.value = sliderB.value;
        updateCalculator();
    }

    function syncFieldsToSliders() {
        sliderA.value = inputA.value;
        sliderB.value = inputB.value;
        updateCalculator();
    }

    // Event Bindings
    formulaInput.addEventListener('input', updateCalculator);
    inputA.addEventListener('input', syncFieldsToSliders);
    inputB.addEventListener('input', syncFieldsToSliders);
    sliderA.addEventListener('input', syncSlidersToFields);
    sliderB.addEventListener('input', syncSlidersToFields);

    // Initial trigger run execution on viewport boot
    updateCalculator();
</script>

</body>
</html>
