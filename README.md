<script>
  let socket;
  let freq = Array(10).fill(0);
  let even = 0, odd = 0;
  let matchStreak = 1, differStreak = 1;
  let maxMatch = 0, maxDiffer = 0;
  let lastDigit = null;

  function startStreaming() {
    const symbol = document.getElementById("symbol").value;
    document.getElementById("status").textContent = `Connecting to ${symbol}...`;

    if (socket) socket.close();

    // Reset stats
    freq = Array(10).fill(0);
    even = 0; odd = 0;
    matchStreak = 1; differStreak = 1;
    maxMatch = 0; maxDiffer = 0;
    lastDigit = null;

    socket = new WebSocket("wss://ws.derivws.com/websockets/v3?app_id=1089");

    socket.onopen = () => {
      document.getElementById("status").textContent = `🔌 Connected: ${symbol}`;
      socket.send(JSON.stringify({ ticks: symbol }));
    };

    socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      if (data.tick && data.tick.quote) {
        const price = data.tick.quote;
        const digit = parseInt(price.toString().slice(-1)); // Get last digit
        if (!isNaN(digit)) {
          updateStats(digit);
        }
      }
    };

    socket.onerror = () => {
      document.getElementById("status").textContent = "❌ Connection error.";
    };
  }

  function updateStats(digit) {
    freq[digit]++;
    digit % 2 === 0 ? even++ : odd++;

    if (lastDigit !== null) {
      if (digit === lastDigit) {
        matchStreak++;
        differStreak = 1;
      } else {
        differStreak++;
        matchStreak = 1;
      }
      maxMatch = Math.max(maxMatch, matchStreak);
      maxDiffer = Math.max(maxDiffer, differStreak);
    }

    lastDigit = digit;
    document.getElementById("liveDigit").textContent = digit;
    renderResults();
  }

  function renderResults() {
    let output = `<div class="grid grid-cols-2 md:grid-cols-5 gap-4 text-center">`;
    for (let i = 0; i <= 9; i++) {
      output += `
        <div class="bg-blue-50 p-4 rounded-lg border border-blue-200">
          <div class="text-xl font-semibold text-blue-800">Digit ${i}</div>
          <div class="text-sm">Frequency: <strong>${freq[i]}</strong></div>
        </div>`;
    }
    output += `</div>`;

    // Predict next likely Match digit if differ streak is long
    let matchPrediction = "";
    if (differStreak >= 4) {
      matchPrediction = `
        <div class="mt-4 bg-purple-50 border border-purple-300 rounded-lg p-4 text-center">
          <p class="text-lg font-semibold text-purple-800">🔮 Prediction: Likely Match Soon</p>
          <p class="text-sm text-gray-600">Try matching on digit: 
            <span class="text-2xl font-bold text-purple-700">${lastDigit}</span> 
            (based on last tick)
          </p>
        </div>`;
    }

    output += `
      <div class="mt-6 p-4 bg-yellow-50 border border-yellow-300 rounded-lg text-sm">
        <p><strong>Even Count:</strong> ${even}</p>
        <p><strong>Odd Count:</strong> ${odd}</p>
        <p><strong>Max Match Streak:</strong> ${maxMatch}</p>
        <p><strong>Max Differ Streak:</strong> ${maxDiffer}</p>
      </div>

      <div class="mt-4 p-4 bg-green-50 border border-green-300 rounded-lg text-sm">
        <h2 class="text-lg font-semibold text-green-800 mb-2">📊 Trading Insights</h2>
        <ul class="list-disc pl-5 space-y-1">
          <li>Current Match Streak: ${matchStreak}</li>
          <li>Current Differ Streak: ${differStreak}</li>
          <li>${matchStreak >= 3 ? "⚠️ Consider a 'Differ' entry soon." : ""}</li>
          <li>${differStreak >= 4 ? "⚠️ Consider a 'Match' entry soon." : ""}</li>
        </ul>
      </div>
      ${matchPrediction}
    `;

    document.getElementById("results").innerHTML = output;
  }
</script>
